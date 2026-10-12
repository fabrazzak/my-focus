# BUILD_NOTES — MyFocus (custom RustDesk) on Windows

Upstream: https://github.com/rustdesk/rustdesk (AGPL-3.0). Built from master commit
`c8532719` (2026-10-10, "Fix/macos mouse release #16516"), submodule `libs/hbb_common` @ `229b9045`.
Versions are taken from `.github/workflows/flutter-build.yml` (job `build-for-windows-flutter`, x86_64)
and `.github/workflows/bridge.yml`.

## Versions used

| Tool | Version | Location |
|---|---|---|
| Windows | 11 Home 10.0.26200, x64 | |
| VS Build Tools 2022 | 17.14, MSVC 14.44.35207, Win SDK 10.0.26100 | pre-installed |
| Python | 3.12.10 | pre-installed |
| Git | 2.48.1 | pre-installed |
| Rust | **1.75.0** (CI `SCITER_RUST_VERSION`), MSVC host, + rustfmt | `D:\dev\rust` |
| cargo-expand | **1.0.95** | `D:\dev\rust\cargo\bin` |
| flutter_rust_bridge_codegen | **1.80.1** (`--features uuid`) | `D:\dev\rust\cargo\bin` |
| LLVM / Clang | **15.0.6** | `D:\dev\LLVM` |
| Flutter | **3.24.5** stable (Dart 3.5.4) + RustDesk custom engine + dropdown patch | `D:\dev\flutter` |
| vcpkg | commit **9e593bb18ea69cc5095e012465dcd675a822ed0d** (2026.07.29), tool 2026-07-27 | `D:\dev\vcpkg` |
| vcpkg pkgs (x64-windows-static) | aom 3.14.1, libvpx 1.15.2, libyuv 1857, opus 1.5.2, libjpeg-turbo 3.2.0, ffmpeg 7.1.1 [amf,nvcodec,qsv], mfx-dispatch 1.35.1, amd-amf 1.4.35, ffnvcodec 12.1.14.0 | `D:\dev\vcpkg\installed` |

Nothing was added to the system PATH. Every session loads the env with:

```powershell
. D:\dev\env.ps1     # sets RUSTUP_HOME, CARGO_HOME, VCPKG_ROOT, LIBCLANG_PATH, PUB_CACHE, PATH
```

`D:\dev\env.ps1`:
```powershell
$env:RUSTUP_HOME   = 'D:\dev\rust\rustup'
$env:CARGO_HOME    = 'D:\dev\rust\cargo'
$env:VCPKG_ROOT    = 'D:\dev\vcpkg'
$env:LIBCLANG_PATH = 'D:\dev\LLVM\bin'
$env:PUB_CACHE     = 'D:\dev\pub-cache'
$env:VCPKG_DEFAULT_HOST_TRIPLET = 'x64-windows-static'
$env:Path = "D:\dev\rust\cargo\bin;D:\dev\LLVM\bin;D:\dev\flutter\bin;D:\dev\vcpkg;$env:Path"
```

## Downloads (all in `D:\dev\downloads`)

| File | URL | SHA256 check |
|---|---|---|
| rustup-init.exe | https://static.rust-lang.org/rustup/dist/x86_64-pc-windows-msvc/rustup-init.exe | matches official `.sha256` (6f4bef66…90bdb7e) |
| LLVM-15.0.6-win64.exe | https://github.com/llvm/llvm-project/releases/download/llvmorg-15.0.6/LLVM-15.0.6-win64.exe | no official checksum; size matches GitHub (290951930). SHA256 22E2F2C3…877AC0A2 |
| flutter_windows_3.24.5-stable.zip | https://storage.googleapis.com/flutter_infra_release/releases/stable/windows/flutter_windows_3.24.5-stable.zip | matches `releases_windows.json` (b8a7485a…d949eeac) |
| windows-x64-release.zip | https://github.com/rustdesk/engine/releases/download/main/windows-x64-release.zip | no official checksum; size matches GitHub (63301215). SHA256 EC8CABF3…A110036E |

## Commands that worked, in order (one-time setup)

```powershell
# 1. Clone
git clone --recurse-submodules https://github.com/rustdesk/rustdesk.git "E:\LOCAL MARKET PLACE\RustDesk\rustdesk"

# 2. Rust 1.75 (no PATH change)
. D:\dev\env.ps1
D:\dev\downloads\rustup-init.exe -y --no-modify-path --profile minimal --default-toolchain 1.75 --default-host x86_64-pc-windows-msvc -c rustfmt

# 3. LLVM 15.0.6 silent install
Start-Process D:\dev\downloads\LLVM-15.0.6-win64.exe -ArgumentList '/S','/D=D:\dev\LLVM' -Wait

# 4. vcpkg at the CI commit
git clone https://github.com/microsoft/vcpkg D:\dev\vcpkg
cd D:\dev\vcpkg; git checkout 9e593bb18ea69cc5095e012465dcd675a822ed0d; .\bootstrap-vcpkg.bat -disableMetrics

# 5. Flutter 3.24.5 (verify SHA256 first) + RustDesk engine + patch
tar.exe -xf D:\dev\downloads\flutter_windows_3.24.5-stable.zip -C D:\dev
flutter config --no-analytics; flutter precache --windows
tar.exe -xf D:\dev\downloads\windows-x64-release.zip -C D:\dev\downloads\rustdesk-engine-x64
Copy-Item -Force D:\dev\downloads\rustdesk-engine-x64\* D:\dev\flutter\bin\cache\artifacts\engine\windows-x64-release
cd D:\dev\flutter; git apply "E:\LOCAL MARKET PLACE\RustDesk\rustdesk\.github\patches\flutter_3.24.4_dropdown_menu_enableFilter.diff"

# 6. Bridge tools
cargo install cargo-expand --version 1.0.95 --locked
cargo install flutter_rust_bridge_codegen --version 1.80.1 --features "uuid" --locked

# 7. vcpkg deps from the repo manifest (~52 min; ffmpeg is the slow one)
cd "E:\LOCAL MARKET PLACE\RustDesk\rustdesk"
vcpkg install --triplet x64-windows-static --x-install-root="D:\dev\vcpkg\installed"
```

## Commands that worked, in order (every build)

```powershell
. D:\dev\env.ps1
cd "E:\LOCAL MARKET PLACE\RustDesk\rustdesk"

# a. Generate the Rust<->Dart bridge (only needed after a fresh clone or a change to src/flutter_ffi.rs)
cd flutter; flutter pub get; cd ..
flutter_rust_bridge_codegen --rust-input ./src/flutter_ffi.rs --dart-output ./flutter/lib/generated_bridge.dart --c-output ./flutter/macos/Runner/bridge_generated.h --llvm-path D:\dev\LLVM

# b. Build (same flags as CI): ~15 min Rust + ~3 min Flutter
python .\build.py --portable --flutter --skip-portable-pack --hwcodec --vram
```

Output: `flutter\build\windows\x64\runner\Release\rustdesk.exe`. It needs the whole `Release` folder (DLLs + `data\`).

**Bridge:** generated with Flutter **3.24.5**, which worked. CI generates it with Flutter 3.22.3 and temporarily
changes `extended_text` 14.0.0 to 13.0.0 in pubspec.yaml. That fallback was **not needed**.

## Errors hit and fixes

| Error | Cause | Fix |
|---|---|---|
| `An Application Control policy has blocked this file. (os error 4551)` on `build-script-build` | Windows 11 **Smart App Control** blocks freshly compiled unsigned exes (every Rust build script) | Windows Security → App & browser control → Smart App Control → **Off**. It took effect after the toggle was set a second time; check with `(Get-MpComputerStatus).SmartAppControlState` (should be `Off`) |
| Built `rustdesk.exe` exits right away with code 1 | Official RustDesk was already running. `main.cpp` finds the existing window with the same app name, brings it to the front and exits | Quit official RustDesk first. After the rebrand, MyFocus uses its own name, so both can run side by side |
| `flutter pub get` modifies `flutter/pubspec.lock` (16 deps) | Committed lock is from a newer Flutter; 3.24.5 pins older SDK packages (async, matcher, meta, path, vm_service…) | Expected. Do **not** commit `pubspec.lock` |
| Log: `printer_driver_adapter.dll not found` | CI downloads the printer driver after the build; this was skipped | Harmless |
| `sed -i` edits converted CRLF to LF in some working files | Git-bash sed | Harmless: `.gitattributes` has `* text=auto`, so git stores LF anyway. Optional: `git checkout -- <file>` after committing restores CRLF |

## Branding (originally branch `custom-brand`, now `main` of the private repo; see "Git repository")

Values: app name **MyFocus**, company **MyFocus**, default icons, Android app ID **com.myfocus.remote**,
server/key **unchanged** (`libs/hbb_common/src/config.rs:117-118`, to be set later).
Reference for which files to patch: rdgen `.github/workflows/generator-windows.yml` (https://github.com/bryangerlach/rdgen).

| File | Change |
|---|---|
| `libs/hbb_common/src/config.rs:72` (submodule) | `APP_NAME` default `"RustDesk"` → `"MyFocus"` (window title, config dir `%APPDATA%\MyFocus`, IPC pipe, service, install dir). UI strings swap "RustDesk" for the app name at runtime (`src/lang.rs:254`), so `src/lang/*.rs` are untouched |
| `Cargo.toml` | `description`, `[package.metadata.winres]` ProductName / FileDescription / OriginalFilename (`MyFocus.exe`) / LegalCopyright |
| `libs/portable/Cargo.toml` | same 5 fields |
| `libs/portable/src/main.rs:21` | `APP_PREFIX` `"rustdesk"` → `"myfocus"` |
| `flutter/windows/runner/Runner.rc:92-98` | CompanyName, FileDescription, InternalName, LegalCopyright, OriginalFilename, ProductName |
| `flutter/lib/desktop/widgets/tabbar_widget.dart:644` | hard-coded `"RustDesk"` title-bar text → `bind.mainGetAppNameSync()` |
| `flutter/lib/desktop/pages/desktop_setting_page.dart:2583` | About: `© <year> MyFocus` + `Based on RustDesk, Copyright © Purslane Tech Pte. Ltd.` |

LegalCopyright wording (keeps the original credit for AGPL):
`Copyright © 2026 MyFocus. Based on RustDesk, Copyright © Purslane Tech Pte. Ltd. (AGPL-3.0)`.
`LICENCE` and source headers are untouched.

### Update check / official download links disabled

| File:line | Change |
|---|---|
| `src/common.rs:1027-1032` | `do_check_software_update()` returns early when `is_custom_client()` is true (app name ≠ "RustDesk"): no call to `api.rustdesk.com`, and `src/updater.rs` can no longer download the official installer |
| `flutter/lib/desktop/pages/desktop_setting_page.dart:485` | "Auto update" checkbox hidden for custom clients (`&& !bind.isCustomClient()`) |

Already off for custom clients, no change needed: the "Check for software update on startup" toggle
(desktop + mobile), the "new version → Download" card (`desktop_home_page.dart:431`), and the mobile
update button (`connection_page.dart:122`). The homepage and privacy links to rustdesk.com remain.

### Git repository
On 2026-10-12 the original `.git` folder (upstream history, both `custom-brand` branches, and the
submodule's repo under `.git\modules`) disappeared from disk before the branding was committed. The
working files were intact. The repo was re-created from the working tree:

- **Remote:** `origin` = https://github.com/fabrazzak/my-focus.git (private, fetch + push), branch `main`.
  There is **no upstream remote** and no RustDesk history: the first commit `e74ea46` is the full tree.
- **hbb_common is vendored**: `libs/hbb_common/` is ordinary files in this repo (not a submodule).
  `.gitmodules` and the stale `libs/hbb_common/.git` pointer file are left on disk but are not tracked.
- **`.gitignore`**: a "MyFocus additions" block was appended. It re-includes `*png/*svg/*jpg`, because
  upstream ignores them and a fresh `git add` would otherwise drop every icon. It also ignores build
  outputs (`/flutter/build/`, `/target/`, `libs/portable/data.bin`, `app_metadata.toml`),
  `flutter/pubspec.lock`, `/.gitmodules`, and keystores/secrets.
- **Fresh clone:** the generated bridge files are ignored, so run step **a** (bridge codegen) before building.
- Commit messages from Windows PowerShell 5.1: pass them with `-m`. Piping a here-string into
  `git commit -F -` adds a UTF-8 BOM to the subject (this happened in `e74ea46`; left as is).

### Updating to a new RustDesk release
Upstream history was dropped, so there is no merge or rebase path. Updates are a **manual file comparison**:
```powershell
# 1. Get the new release next to this repo (not inside it)
git clone --recurse-submodules --branch <tag> https://github.com/rustdesk/rustdesk.git D:\dev\rustdesk-<tag>
# 2. See what MyFocus changed vs the base it was built from (upstream c8532719, hbb_common 229b9045):
#    clone that base too, then compare file by file
git clone --recurse-submodules https://github.com/rustdesk/rustdesk.git D:\dev\rustdesk-base
git -C D:\dev\rustdesk-base checkout c8532719; git -C D:\dev\rustdesk-base submodule update
git diff --no-index D:\dev\rustdesk-base\src "E:\LOCAL MARKET PLACE\RustDesk\rustdesk\src"   # repeat per folder
# 3. Copy the new release over this working tree (keep .git, .gitignore, BUILD_NOTES.md, CODEBASE_REPORT.md),
#    then re-apply the branding table above by hand. Search terms:
#    "RustDesk Remote Desktop", "Purslane", APP_NAME, APP_PREFIX, showAutoUpdate, do_check_software_update
# 4. Rebuild, test, commit to main, push to origin.
```

## Rebuild with branding (2026-10-12): success
```powershell
. D:\dev\env.ps1; cd "E:\LOCAL MARKET PLACE\RustDesk\rustdesk"
python .\build.py --portable --flutter --skip-portable-pack --hwcodec --vram
Rename-Item flutter\build\windows\x64\runner\Release\rustdesk.exe MyFocus.exe
cd libs\portable; pip install -r requirements.txt
python .\generate.py -f ..\..\flutter\build\windows\x64\runner\Release -o . -e ..\..\flutter\build\windows\x64\runner\Release\MyFocus.exe
Copy-Item ..\..\target\release\rustdesk-portable-packer.exe ..\..\target\release\MyFocus-1.5.0-install.exe
```
- `build.py`: exit 0. Rust release build **6m 23s** (16 upstream warnings, all unused-code/naming), Flutter Windows build **2m 23s**.
- Installer: `generate.py` exit 0 (it runs `cargo build --locked --release` itself). Output `target\release\MyFocus-1.5.0-install.exe` (25 MB).
- Tested OK: window title, taskbar, tray, About page, exe properties, installer name, runs alongside official RustDesk.
- `generate.py` leaves `libs\portable\data.bin` and `app_metadata.toml` behind (ignored by git).

## Still to do (status at time of writing)
1. ~~Rebuild with branding, rename to `MyFocus.exe`~~: done (see above).
2. ~~Self-extracting installer~~: done.
3. ~~Check branding in title, taskbar, tray, About, installer~~: done.
4. ~~Commit~~: done, as a fresh repo pushed to the private origin (see "Git repository").
5. ~~Git remotes~~: now a private GitHub `origin` only (see "Git repository").
6. Android APK (arm64-v8a) in WSL Ubuntu, following CI job `build-rustdesk-android`. **On hold:** Android feature changes to be discussed first. Android branding (app ID, label, Kotlin package, strings) is not applied yet; see CODEBASE_REPORT.md §4.
7. Icons are still RustDesk's (Windows and Android).
