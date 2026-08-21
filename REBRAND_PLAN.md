# Aurora Files — Rebrand & CI Plan

Repo: `aurora-files-client` (clone of nextcloud/desktop, `stable-34.0-qt-6.10.2` craft branch)

## Decisions
- CI check: verify via a real PR (#1, branch `ci/verify-windows-build`) since `windows-build-and-test.yml` only triggers on `pull_request`.
- Rebrand scope: full — app name, executable, icons, vendor/domain, and server URL.
- Server URL: locked/enforced (`APPLICATION_SERVER_URL_ENFORCE=ON`).
- Branding approach: edit defaults directly in `NEXTCLOUD.cmake` / `CMakeLists.txt` (this is our own fork, not consuming upstream craft-blueprints overrides). No `-D` flags needed in CI/craftmaster.ini.
- Updater: `BUILD_UPDATER` disabled for now; re-enable once a self-hosted update server exists.
- Non-Windows CI workflows (Linux/macOS builds) temporarily relocated to `unused_workflows/` to avoid runner cost while we focus on Windows; Windows builds to move to self-hosted runners (see below).

## Branding values applied
- `APPLICATION_NAME` = "Aurora Files"
- `APPLICATION_SHORTNAME` = "AuroraFiles"
- `APPLICATION_EXECUTABLE` = "AuroraFiles" (bare base name; CMake/platform appends `.exe` on Windows automatically)
- `APPLICATION_ICON_NAME` = "AuroraFiles"
- `APPLICATION_DOMAIN` = "auroratechnology.co.uk"
- `APPLICATION_VENDOR` = "Aurora Technology (OTGH) Ltd"
- `APPLICATION_SERVER_URL` = "https://collab.otgh.cloud"
- `APPLICATION_SERVER_URL_ENFORCE` = ON
- `APPLICATION_REV_DOMAIN` = "uk.co.auroratechnology.desktopclient"
- `APPLICATION_REV_DOMAIN_DBUS` = "desktopclient.auroratechnology.co.uk"
- `BUILD_UPDATER` = OFF
- Windows shell-extension/MSI GUIDs and `APPLICATION_URI_HANDLER_SCHEME` regenerated (see `NEXTCLOUD.cmake` / `CMakeLists.txt`) to avoid colliding with a stock Nextcloud client installed on the same machine.
- `theme/colored/AuroraFiles-icon.svg`, `AuroraFiles-icon-square.svg`, `AuroraFiles-macOS-icon.svg`, `AuroraFiles-w10startmenu.svg` created as duplicates of the stock Nextcloud SVGs — placeholders until real artwork is supplied.

## Key research findings
- `NEXTCLOUD.cmake` is the single central branding file: `APPLICATION_NAME`/`SHORTNAME`/`EXECUTABLE`/`ICON_NAME`/`DOMAIN`/`VENDOR`/`SERVER_URL`/`REV_DOMAIN`/colors/GUIDs.
- `config.h.in` already has `#cmakedefine APPLICATION_SERVER_URL` / `_ENFORCE` passthrough (no edit needed).
- `src/libsync/theme.cpp:450-456` reads `APPLICATION_SERVER_URL` / `_ENFORCE` at startup → `setOverrideServerUrl()`; consumed by `src/gui/wizard/accountwizardcontroller.cpp:116-132` (pre-fills + disables URL field when enforced).
- `CMakeLists.txt:31-48` defines CFAPI shell extension GUIDs/display names (uses `APPLICATION_NAME` already) and `APPLICATION_URI_HANDLER_SCHEME` (was `"nc"`).
- `admin/win/msi/OEM.wxi.in` and `mirall.desktop.in` already use `@APPLICATION_*@` placeholders — no edits needed there.
- `craftmaster.ini`: just craft config (Qt 6.10.2, blueprint revisions stable-34.0), no branding vars — no changes needed since branding is baked into repo defaults.
- macOS icon logic in `NEXTCLOUD.cmake` already has a branch for "branded customer builds" (checks `APPLICATION_NAME STREQUAL "Nextcloud"` to decide Icon Composer vs custom SVG icon) — our custom `AuroraFiles-icon.svg` automatically uses the legacy SVG/ECM pipeline.
- `admin/linux/build-appimage.sh` referenced by `linux-appimage.yml` does not exist in this clone (not a `.gitignore` exclusion — genuinely missing from source). Not a blocker since that workflow is now parked in `unused_workflows/`, but flag if Linux builds are revisited later.

## Steps

### Phase A — CI verification (done)
1. Opened PR #1 (`ci/verify-windows-build`) against `main` with a trivial change to trigger `windows-build-and-test.yml` on unmodified code.

### Phase B — Core branding edits (done)
2. Edited `NEXTCLOUD.cmake` with all values above.
3. Edited `CMakeLists.txt` (CFAPI GUIDs + URI scheme).
4. Duplicated stock icon SVGs under `AuroraFiles-*` filenames.
5. Committed and pushed to PR #1 branch to re-trigger CI with branding applied.

### Phase C — CI cost control (in progress)
6. Moved Linux/macOS build & test workflows (`linux-appimage*.yml`, `linux-clang-compile-tests.yml`, `linux-gcc-compile-tests.yml`, `macos-build-and-test.yml`, `macos-craft-ci.yml`, `nextcloudfileproviderkit.yml`, `nextcloudcmd-provisioning-tests.yml`) into `unused_workflows/` so they no longer trigger. Lightweight lint/bot workflows (clang-format, clang-tidy-review, check-translations, qml-label-check, sonarcloud, reuse, bug-report-labeler, needsinfohelper, pr-feedback, stale, block-unconventional-commits, fixup) were left in place — cheap `ubuntu-latest` jobs, not full builds.
7. Migrate `windows-build-and-test.yml` to run on a self-hosted Windows runner instead of GitHub-hosted `windows-2022`, to avoid GitHub Actions billing for expensive Craft/MSVC builds (see separate self-hosted runner outline).

### Phase D — Build & verify (pending)
8. Confirm Windows CI (on self-hosted runner once migrated) succeeds with branding changes.
9. Manual smoke test of built installer: correct app name, wizard pre-fills/locks the URL, no GUID/URI-scheme collisions with a real Nextcloud client installed side by side, updater UI absent/disabled.

## Relevant files
- `NEXTCLOUD.cmake` — central branding vars, GUIDs, `BUILD_UPDATER`
- `CMakeLists.txt` (lines ~31-48) — CFAPI GUIDs + URI scheme
- `theme/colored/` — icon file duplication
- `.github/workflows/windows-build-and-test.yml` — CI to verify / migrate to self-hosted
- `unused_workflows/` — parked Linux/macOS workflows

## Further considerations
1. Real artwork still needed for `AuroraFiles-*` icon/logo assets (currently stock duplicates).
2. Re-enable `BUILD_UPDATER` once a self-hosted update server is available, and point `APPLICATION_UPDATE_URL` at it.
3. `admin/linux/build-appimage.sh` is missing from source — needed if/when Linux builds are re-enabled.
