# CODEBASE_REPORT — MyFocus (custom RustDesk)

Snapshot of branch `custom-brand` (upstream master `c8532719`, `libs/hbb_common` @ `229b9045`), 2026-10-12.
Read-only survey to help decide what to customise. Line numbers refer to this snapshot and drift on rebase.
See `BUILD_NOTES.md` for build steps.

---

## 1. High-level architecture

One Rust crate (`librustdesk`) holds both the **controlled side** ("server": capture, input, the connection
handler) and the **controlling side** ("client": session, decoding). Flutter is only the UI. It calls Rust
through a generated FFI bridge and gets events back as streams.

```
            ┌──────────────────────────── Flutter UI (Dart) ─────────────────────────────┐
            │ flutter/lib/desktop/*   flutter/lib/mobile/*   flutter/lib/common/*, models/* │
            └───────────────▲───────────────────────────────────────────▲─────────────────┘
       bind.xxx() calls     │ generated_bridge.dart                     │ MethodChannel "mChannel"
       + event streams      │ (flutter_rust_bridge 1.80)                │ (Android only)
            ┌───────────────┴──────────────┐              ┌─────────────┴─────────────────┐
            │ src/flutter_ffi.rs  (API)    │              │ Kotlin: MainActivity,         │
            │ src/flutter.rs (sessions →   │◄──── JNI ───►│ MainService (MediaProjection),│
            │   Dart events)               │  ffi.kt /    │ InputService (Accessibility)  │
            └───────────────┬──────────────┘  scrap/     └───────────────────────────────┘
                            │                 android/ffi.rs
   ┌────────────────────────┴──────────────── Rust core (src/) ─────────────────────────────┐
   │ client.rs, ui_session_interface.rs   ← controlling side (connect, decode, send input)  │
   │ server.rs, server/connection.rs      ← controlled side (auth, permissions, services)   │
   │ server/video_service.rs  → libs/scrap   (capture + VP8/VP9/AV1/H26x encode)             │
   │ server/input_service.rs  → libs/enigo   (desktop input injection)                       │
   │ rendezvous_mediator.rs   ← registers ID with ID server, hole-punch / relay              │
   │ ipc.rs                   ← desktop: UI ⇄ background service process                    │
   └────────────────────────┬────────────────────────────────────────────────────────────────┘
                            │
   ┌────────────────────────┴───── libs/hbb_common (git submodule, shared with rustdesk-server)┐
   │ config.rs (APP_NAME, server list, public key, Config/LocalConfig files), protobuf          │
   │ rendezvous + message protos, TCP/UDP/WebSocket/proxy sockets, TLS                          │
   └────────────────────────────────────────────────────────────────────────────────────────────┘
                            │ network
                 ID/rendezvous server (hbbs :21116) + relay (hbbr :21117)
```

- **Windows:** `MyFocus.exe` is the Flutter runner. It loads `librustdesk.dll` ([native_model.dart:142](flutter/lib/models/native_model.dart#L142)). When installed, a Windows service runs `--service`, which starts a `--server` process in the user session. The UI talks to that process over named-pipe IPC ([src/ipc.rs](src/ipc.rs)).
- **Android:** a single process. Kotlin loads `librustdesk.so`, Flutter uses the same library through the bridge, and screen frames and input go Kotlin ⇄ Rust over JNI.

## 2. Folder map

| Folder | What it holds |
|---|---|
| `src/` | The main Rust crate: client, server, IPC, FFI, platform code, translations (`src/lang/*.rs`) |
| `src/server/` | Controlled-side services: video, audio, clipboard, input, terminal, printer, `connection.rs` (per-peer handler) |
| `src/platform/` | OS-specific glue: `windows.rs` (install, service, shortcuts, registry), `linux.rs`, `macos.rs` |
| `src/client/` | Controlling-side helpers: `io_loop.rs` (session loop), audio playback, file transfer |
| `src/privacy_mode/` | Windows/macOS "privacy mode" (black screen on the remote) |
| `src/ui/` | Legacy Sciter UI. Not used by the Flutter build |
| `flutter/lib/` | All UI code (Dart): `desktop/`, `mobile/`, `common/` (shared widgets), `models/` (state + bridge calls) |
| `flutter/android/` | Android app shell: Gradle, manifest, Kotlin services, resources/icons |
| `flutter/windows/` | Windows Flutter runner (C++): `runner/main.cpp`, `Runner.rc` (exe metadata), app icon |
| `libs/hbb_common/` | **Submodule.** Config, protocol, networking (shared with the server). Changing it means a second commit |
| `libs/base/` | Client-only shared code: option keys (`config/keys.rs`), file transfer, message proto |
| `libs/scrap/` | Screen capture (DXGI/GDI on Windows; Android frames via JNI) and video codecs |
| `libs/enigo/` | Desktop keyboard/mouse injection (Windows/macOS/Linux) |
| `libs/clipboard/` | Clipboard and file-copy-paste |
| `libs/portable/` | Self-extracting installer/packer (`generate.py` → `rustdesk-portable-packer.exe`) |
| `libs/virtual_display/`, `libs/remote_printer/` | Windows virtual display driver glue, remote printing |
| `res/` | Icons (`icon.ico`, `tray-icon.ico`, PNGs), Linux packaging, admin-API helper scripts |
| `build.py` | Build driver (Rust + Flutter, Windows packaging) |
| `.github/` | Upstream CI workflows (reference for tool versions) and the Flutter dropdown patch |

## 3. Key files

**Entry points**
- [flutter/lib/main.dart:38](flutter/lib/main.dart#L38): `main()` picks the mode: `runMobileApp()` (L176), `runMainApp()` (L133, desktop main window), `runMultiWindow()` (remote/file/terminal tabs), `runConnectionManagerScreen()` (L285, the "someone is connected" window), `runInstallPage()`.
- [flutter/windows/runner/main.cpp:83-93](flutter/windows/runner/main.cpp#L83-L93): gets the app name from Rust and brings an already-running window with the same name to the front (this is why two copies with the same name can't run side by side).
- [src/core_main.rs](src/core_main.rs): desktop command-line dispatch: `--service`, `--server`, `--tray`, `--cm`, `--install`, `--silent-install`, `--uninstall`, `--config`, `--password`, `--get-id`, `--import-config`.
- [src/flutter_ffi.rs](src/flutter_ffi.rs): every function Dart can call (`main_*`, `session_*`, `cm_*`), plus the Android JNI entry points `Java_ffi_FFI_startServer` / `startService` (L2953-L2985).

**Connection and signalling**
- [src/rendezvous_mediator.rs](src/rendezvous_mediator.rs): `start_all()` (L274) registers this device's ID with the ID server, keeps it alive, and handles punch-hole/relay requests.
- [src/client.rs](src/client.rs): controlling side. Resolves the peer ID, NAT traversal or relay, key exchange, then the session loop in [src/client/io_loop.rs](src/client/io_loop.rs).
- [src/server.rs](src/server.rs): `start_server()` (L659) and the service registry. [src/server/connection.rs](src/server/connection.rs) handles one incoming peer: password/2FA auth, permission flags (keyboard, clipboard, file, audio…), and routing messages to services.
- [src/ipc.rs](src/ipc.rs): desktop IPC between the UI, the service and the connection manager (config sync, CM events).
- [src/hbbs_http/](src/hbbs_http/): optional HTTP API (login, address book, audit). Only used with an API server.

**Screen capture and video**
- [src/server/video_service.rs](src/server/video_service.rs): capture loop, encoder selection, QoS ([video_qos.rs](src/server/video_qos.rs)).
- `libs/scrap/src/common/dxgi.rs` and `libs/scrap/src/dxgi/`: Windows capture. `libs/scrap/src/common/android.rs` + [libs/scrap/src/android/ffi.rs](libs/scrap/src/android/ffi.rs): Android frames pushed from Kotlin (`onVideoFrameUpdate`, L125).
- Codecs: `libs/scrap/src/common/{vpxcodec,aom,hwcodec,vram,mediacodec}.rs`.

**Input injection**
- Desktop: [src/server/input_service.rs](src/server/input_service.rs) → `libs/enigo/src/win/win_impl.rs` (Windows `SendInput`). Keyboard mapping: [src/keyboard.rs](src/keyboard.rs).
- Android: [connection.rs:3167](src/server/connection.rs#L3167) → `call_main_service_pointer_input` ([scrap/src/android/ffi.rs:295](libs/scrap/src/android/ffi.rs#L295)) → JNI → `MainService.rustPointerInput` → `InputService.kt` (AccessibilityService gestures). Keys: `call_main_service_key_event` → `KeyboardKeyEventMapper.kt`.

**Config, server and key**
- [libs/hbb_common/src/config.rs](libs/hbb_common/src/config.rs): `APP_NAME` (L72), `ORG` (L57, `com.carriez`, used for the macOS/Linux config path), **`RENDEZVOUS_SERVERS` (L117) and `RS_PUB_KEY` (L118)**: put your own hbbs host and key here to hard-wire a self-hosted server. Also holds the `Config` (ID, password, key pair), `Config2` and `LocalConfig` structs, stored as `%APPDATA%\MyFocus\config\MyFocus*.toml`.
- [libs/base/src/config/keys.rs](libs/base/src/config/keys.rs): all option-key names (`enable-keyboard`, `custom-rendezvous-server`, `relay-server`, `key`, …).
- [src/custom_server.rs](src/custom_server.rs): reads `host=…,key=…` from the exe file name or a config string (the no-recompile way to set a server).
- [src/common.rs](src/common.rs): `get_app_name()` (L1081), `is_custom_client()` (L2559), `do_check_software_update()` (L1026, early return added).

## 4. Where the branding lives

**Already changed on `custom-brand`** (uncommitted; from BUILD_NOTES.md and `git status`):

| File | Controls |
|---|---|
| `libs/hbb_common/src/config.rs:72` (submodule) | `APP_NAME = "MyFocus"`: window title, config dir, IPC pipe name, service name, install dir, and "RustDesk" text in translations (swapped at runtime by `src/lang.rs`) |
| `Cargo.toml` | Crate description + `[package.metadata.winres]` (exe Properties → Details for the Rust binary) |
| `libs/portable/Cargo.toml` | Same metadata for the installer exe |
| `libs/portable/src/main.rs:21` | `APP_PREFIX = "myfocus"`: installer's extraction folder/name prefix |
| `flutter/windows/runner/Runner.rc` | `MyFocus.exe` file metadata (company, product, description, copyright) |
| `flutter/lib/desktop/widgets/tabbar_widget.dart` | Title-bar text now from `bind.mainGetAppNameSync()` instead of hard-coded "RustDesk" |
| `flutter/lib/desktop/pages/desktop_setting_page.dart` | About page copyright + "Based on RustDesk" credit; "Auto update" checkbox hidden for custom clients |
| `src/common.rs` | Update check disabled for custom clients (no call to api.rustdesk.com) |

Not branding, also modified: `flutter/pubspec.lock` (from `flutter pub get`; leave uncommitted). New file: `BUILD_NOTES.md`.

**Not yet branded (still RustDesk):**
- Icons: `res/icon.ico`, `res/tray-icon.ico`, `res/*.png`, `flutter/windows/runner/resources/app_icon.ico`, Android `res/mipmap-*`.
- The exe is still built as `rustdesk.exe` (renamed after the build), and the Flutter side still loads `librustdesk.dll`.
- **Android, none of it applied yet:** `applicationId`/`namespace` `com.carriez.flutter_hbb` ([build.gradle:85,102](flutter/android/app/build.gradle#L85)); the Kotlin package folder `com/carriez/flutter_hbb`; manifest `android:label="RustDesk"` / `"RustDesk Input"` ([AndroidManifest.xml:32,52](flutter/android/app/src/main/AndroidManifest.xml#L32)); deep-link scheme `rustdesk` (L82); `res/values/strings.xml` (app name, accessibility description); hard-coded strings in `BootReceiver.kt:39`, `MainService.kt:50,819-825` (notification), `FloatingWindowService.kt:306`.
- Links to rustdesk.com on the mobile settings/About page (`mobile/pages/settings_page.dart:39,1033,1066,1179`) and `mobile/pages/connection_page.dart:127`.

## 5. Where features live

| Feature | Windows / desktop | Android |
|---|---|---|
| **Service auto-start** | `install_me()` [windows.rs:1593](src/platform/windows.rs#L1593) creates the service; `install_service()` L3349; `start_os_service()` L556 is what the service runs (`--service`); tray: [src/tray.rs:12](src/tray.rs#L12). UI: `desktop/pages/install_page.dart`, plus the install/service controls in `desktop_home_page.dart` | [BootReceiver.kt](flutter/android/app/src/main/kotlin/com/carriez/flutter_hbb/BootReceiver.kt) (needs the "start on boot" pref + battery-opt + overlay permissions); toggle in [mobile settings_page.dart:600](flutter/lib/mobile/pages/settings_page.dart#L600) |
| **Permissions flow** | Per-peer rights in `src/server/connection.rs`; options under Settings → Security in `desktop_setting_page.dart`; accept/deny window in `desktop/pages/server_page.dart` (connection manager) | Dart: `checkAndroidPermission()` / `toggleService()` in [server_model.dart:202,377](flutter/lib/models/server_model.dart#L202); UI `PermissionChecker` in [mobile server_page.dart:568](flutter/lib/mobile/pages/server_page.dart#L568). Kotlin: `MainActivity.kt` method channel (L266: `init_service`, `check_permission`, `request_permission`, `start_capture`…), `PermissionRequestTransparentActivity.kt` (screen-capture consent), `InputService.kt` (accessibility = remote input) |
| **Main screens / buttons** | `desktop/pages/desktop_home_page.dart` (left pane: ID, password, service status), `desktop/pages/connection_page.dart` (connect box + recent peers), `desktop/widgets/remote_toolbar.dart` (in-session toolbar), `desktop/pages/remote_page.dart` (remote view), `common/widgets/peer_*.dart` (peer cards/tabs) | `mobile/pages/home_page.dart` (bottom tabs), `connection_page.dart` (connect), `server_page.dart` ("Share screen" tab: start/stop service, permissions, connected clients), `remote_page.dart` (controlling another device) |
| **Settings** | `desktop/pages/desktop_setting_page.dart` (all tabs: General, Security, Network, Display, Account, About) | `mobile/pages/settings_page.dart` |
| **Option storage** | Keys in `libs/base/src/config/keys.rs`; read/write via `bind.mainGetOption` / `mainSetOption` → `src/ui_interface.rs` → `hbb_common::config` | Same, plus Android SharedPreferences for boot settings (`common.kt`) |
| **Android native code** | — | `flutter/android/app/src/main/kotlin/com/carriez/flutter_hbb/`: `MainActivity.kt` (Flutter ⇄ native channel), `MainService.kt` (foreground service, MediaProjection → ImageReader → Rust; notifications), `InputService.kt` (accessibility input), `FloatingWindowService.kt` (floating overlay icon), `AudioRecordHandle.kt`, `RdClipboardManager.kt`, `VolumeController.kt`. JNI declarations: [kotlin/ffi.kt](flutter/android/app/src/main/kotlin/ffi.kt). Rust side: [libs/scrap/src/android/ffi.rs](libs/scrap/src/android/ffi.rs) + the `Java_ffi_FFI_*` functions in `src/flutter_ffi.rs` |

## 6. Android vs Windows split

**Shared (one code base for both):**
- The whole Rust core: protocol, `rendezvous_mediator.rs`, `client.rs`, `server/connection.rs`, video/audio services, codecs, config (`hbb_common`), translations.
- Flutter `common/`, `models/` (`model.dart`, `server_model.dart`, `input_model.dart`, …) and `generated_bridge.dart`.

**Windows / desktop only:**
- Flutter `lib/desktop/**`, the `flutter/windows/` runner.
- Rust `src/platform/windows.rs`, `src/tray.rs`, `src/ipc.rs` (multi-process model), `src/core_main.rs` CLI, `src/privacy_mode/win_*`, `src/server/{input_service,portable_service,printer_service,terminal_*}.rs`, `libs/enigo`, DXGI capture in `libs/scrap`, `libs/virtual_display`, `libs/portable`.

**Android only:**
- Flutter `lib/mobile/**` (also used on iOS).
- `flutter/android/**` (Kotlin, manifest, Gradle, resources).
- Rust `libs/scrap/src/android/` (JNI glue, frame intake) and `libs/scrap/src/common/{android,mediacodec}.rs`; `Java_ffi_FFI_*` in `src/flutter_ffi.rs`; `#[cfg(target_os = "android")]` branches in `connection.rs` (input → JNI) and `server.rs` (`input_service` is stubbed out).

**Implications for customising:**
- Behaviour changes in the Rust core or Flutter `common/`/`models/` affect **both** apps.
- Phone-side features (what the phone does when you control it: permissions, boot start, notifications, floating icon, capture) live mostly in **Kotlin + `mobile/pages/server_page.dart` + `server_model.dart`**.
- PC-side controlling features (toolbar, remote view, keyboard mapping) live in **`desktop/pages/remote_page.dart`, `desktop/widgets/remote_toolbar.dart`, `src/client.rs`, `src/keyboard.rs`**.
- Changes in `libs/hbb_common` need a separate submodule commit. Prefer `libs/base` or `src/` for anything client-only.
