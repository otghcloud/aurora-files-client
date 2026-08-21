# Windows Server 2022 Self-Hosted Runner — Manual VM Setup

Goal: a Windows Server 2022 VM (on your Proxmox VE infrastructure) that can run the
`Windows Build and Test` workflow ([.github/workflows/windows-build-and-test.yml](.github/workflows/windows-build-and-test.yml))
as a GitHub Actions self-hosted runner, without depending on GitHub-hosted `windows-2022` runners.

This VM does **not** need to be an exact clone of GitHub's hosted image — the workflow itself
installs most of its own tooling (Craft, OpenCppCoverage, Inkscape via Chocolatey, Python via
`actions/setup-python`). This VM only needs the baseline platform tooling that the workflow and
Craft assume already exists on the machine.

---

## 1. Provision the VM in Proxmox

1. Create a new VM in Proxmox VE:
   - **OS**: Windows Server 2022 (attach your Server 2022 ISO as the CD/DVD drive).
   - **Machine type**: `q35` (recommended for modern UEFI/Windows guests).
   - **BIOS**: OVMF (UEFI) + add an EFI disk.
   - **SCSI Controller**: `VirtIO SCSI single` for best disk performance.
   - **Disk**: at least 120 GB (Craft's dependency cache + Qt6 + VS toolchain + build artifacts add up quickly). Use VirtIO block if you're comfortable injecting drivers during install (see below), otherwise SATA is simpler for the initial install.
   - **CPU**: 4+ vCPUs (host type, so instruction sets like AVX2 are exposed — Qt/VS builds benefit).
   - **Memory**: 16 GB minimum, 32 GB preferred (Craft + MSVC + Qt6 + Ninja parallel builds are memory-hungry).
   - **Network**: VirtIO network adapter.
2. Attach the Proxmox **VirtIO driver ISO** (`virtio-win.iso`, available from the [Fedora virtio-win project](https://github.com/virtio-win/virtio-win-pkg-scripts) or bundled with Proxmox) as a second CD/DVD drive — needed if you use VirtIO disk/network during Windows Setup, so you can load drivers when prompted.
3. Boot the VM and run through Windows Setup as normal (Standard or Datacenter, Desktop Experience). Load VirtIO drivers from the attached driver ISO at the disk-selection screen if using VirtIO disk.
4. After install, install the VirtIO **guest tools** (from the same ISO, `virtio-win-guest-tools.exe`) for balloon memory driver, QEMU guest agent, etc. Enable the QEMU Guest Agent in the VM's Proxmox options afterwards.
5. Apply Windows Updates fully and reboot as needed.

---

## 2. Base OS configuration

1. Set a static IP (or a DHCP reservation) so the runner is reliably reachable for maintenance.
2. Create a dedicated local (non-admin, where possible) user account to run the Actions runner service, e.g. `svc-gh-runner`. Note: the runner's Windows service install step (below) requires an interactive/administrator PowerShell session to *install*, but the service itself can run under a lower-privileged account if you configure that in the service afterwards (`services.msc` → Log On tab).
3. Disable/uninstall anything not required (IE Enhanced Security Configuration is off by default on Server Core only — on Desktop Experience, disable it via Server Manager > Local Server > IE Enhanced Security Configuration, since Craft/Inkscape/choco downloads will otherwise be blocked by prompts).
4. Set the PowerShell execution policy to allow local scripts:
   ```powershell
   Set-ExecutionPolicy RemoteSigned -Scope LocalMachine -Force
   ```

---

## 3. Install required software

Install everything below using an elevated (Administrator) PowerShell / Command Prompt session.

### 3.1 Chocolatey (package manager)

The workflow itself installs `OpenCppCoverage` and `inkscape` via `choco install`, so Chocolatey
just needs to exist on the box once:

```powershell
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
```

### 3.2 Git

```powershell
choco install git -y --params "'/GitAndUnixToolsOnPath'"
```

### 3.3 PowerShell 7 (`pwsh`)

The workflow's steps use `shell: pwsh`. Windows Server 2022 only ships Windows PowerShell 5.1 by
default — PowerShell 7 must be installed explicitly:

```powershell
choco install powershell-core -y
```

Restart the shell/session afterwards so `pwsh` is on `PATH`.

### 3.4 Python 3.12 (x64)

Craft is a Python-based tool. Install the same version the workflow's hosted-runner path uses:

```powershell
choco install python312 -y
```

Confirm with `python --version` (expect `3.12.x`) in a new shell.

### 3.5 Visual Studio 2022 Build Tools (C++ workload)

Craft auto-detects the MSVC toolchain (`windows-msvc2022_64-cl` target in [craftmaster.ini](craftmaster.ini)).
You need the **Desktop development with C++** workload — the full Visual Studio IDE is not required,
Build Tools are sufficient and much smaller:

```powershell
choco install visualstudio2022buildtools -y --package-parameters "--add Microsoft.VisualStudio.Workload.VCTools --add Microsoft.VisualStudio.Component.Windows11SDK.22621 --add Microsoft.VisualStudio.Component.VC.ATL --add Microsoft.VisualStudio.Component.VC.ATLMFC --includeRecommended"
```

This installs, among other things, `vcvarsall.bat` (used in the craft-blueprints README's manual
build instructions) and the MSVC v143 x64/x86 build tools + Windows 11 SDK.

> If you'd rather use the full Visual Studio 2022 Community IDE (useful if you also want to debug
> interactively on this box), install `visualstudio2022community` instead and select the "Desktop
> development with C++" workload in the Visual Studio Installer GUI.

### 3.6 Visual C++ Redistributable

The Actions runner and some build tooling expect the latest VC++ runtime to be present:

```powershell
choco install vcredist140 -y
```

### 3.7 Inkscape and OpenCppCoverage (optional to pre-install)

The workflow installs these itself on every run via `choco install inkscape` /
`choco install opencppcoverage`. You can leave this to the workflow (simplest), or pre-install them
here to shave a little time off every build:

```powershell
choco install inkscape -y
choco install opencppcoverage -y
```

---

## 4. Install the GitHub Actions self-hosted runner

1. On GitHub: go to the repository → **Settings → Actions → Runners → New self-hosted runner**,
   choose **Windows** / **x64**. GitHub will show you a registration token and download URL —
   these are time-limited and unique per registration, so use the ones shown to you at the time.
2. In an elevated PowerShell prompt on the VM:
   ```powershell
   mkdir C:\actions-runner ; cd C:\actions-runner
   Invoke-WebRequest -Uri <the actions-runner-win-x64-*.zip URL from GitHub> -OutFile actions-runner.zip
   Expand-Archive -Path actions-runner.zip -DestinationPath . -Force
   ```
3. Register the runner (values from the GitHub UI):
   ```powershell
   ./config.cmd --url https://github.com/otghcloud/aurora-files-client --token <REGISTRATION_TOKEN> --labels self-hosted,windows,aurora-onprem --name aurora-w22-runner-01
   ```
4. When prompted "Would you like to run the runner as a service?", answer **yes** — this installs
   it as a Windows service so it survives reboots and doesn't need an interactive session.
5. Verify the service is running:
   ```powershell
   Get-Service actions.runner.*
   ```
   You should see it `Running`. In the GitHub UI (Settings → Actions → Runners) the runner should
   show as **Idle**.

### Service account note
By default the runner service installs under the account you registered it as (often
`NT AUTHORITY\SYSTEM` if run from an admin session, or the logged-in user). For better isolation,
consider creating the dedicated `svc-gh-runner` local account mentioned in step 2 above and
switching the service's "Log on as" identity via `services.msc`, granting it "Log on as a service"
rights (`secpol.msc` → Local Policies → User Rights Assignment).

---

## 5. Point the workflow at this runner

Once the runner shows **Idle** in GitHub, update [.github/workflows/windows-build-and-test.yml](.github/workflows/windows-build-and-test.yml):

```yaml
runs-on: [self-hosted, windows, aurora-onprem]
```

(replacing `runs-on: windows-2022`). Do this as a separate, deliberate commit once you've confirmed
the runner is online — don't push it until you're ready, to avoid queuing jobs against an
unregistered/offline runner.

---

## 6. Security notes (read before enabling)

- **Only use this self-hosted runner with private repositories**, or with strict branch/PR
  controls. Any workflow triggered by a `pull_request` from an external contributor executes their
  code on this VM. GitHub's own guidance: forks of public repos can run arbitrary code on
  self-hosted runners via a crafted PR.
- Keep the runner's registration scoped to this one repository (not organization-wide) unless you
  need to share it, to limit blast radius.
- GitHub-hosted runners are ephemeral (destroyed after each job); this VM is persistent. Consider:
  - Periodically re-imaging/snapshot-reverting the VM (Proxmox snapshots make this easy) to clear
    any state left by previous builds.
  - Optionally running the runner with `--ephemeral` (deregisters after each job; you'd then need
    automation — e.g. a Proxmox snapshot-revert + re-register script — to bring it back for the
    next job).
- Patch the OS and Chocolatey packages regularly; this machine has outbound internet access
  (needed for GitHub, Craft's dependency downloads, and Chocolatey) so treat it like any other
  internet-facing build server.

---

## 7. Validating the setup

Before relying on this runner for real PRs, do a manual smoke test:

```powershell
git --version
pwsh --version
python --version
choco --version
& "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\VC\Auxiliary\Build\vcvarsall.bat" x64
```

All should succeed without errors. Then trigger a PR against `ci/verify-windows-build` (or open a
fresh no-op PR) once the workflow's `runs-on` has been updated, and watch **Settings → Actions →
Runners** to confirm the job is picked up by `aurora-w22-runner-01` rather than a GitHub-hosted
runner.
