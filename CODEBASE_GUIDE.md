# ZL2NFS (Diavlon Launcher XL) — Codebase Guide for AI Agents

> **Purpose**: This document gives any AI agent full context to work on this codebase without starting from zero.

---

## 1. Project Identity

| Field | Value |
|---|---|
| **Name** | Diavlon Launcher XL (DLXL) |
| **Repo** | `https://github.com/Con-Artist-1/ZL2NFS.git` (this fork) |
| **Upstream fork of** | `https://github.com/Star1xr/ZalithLauncher2Plus` |
| **Original upstream** | `https://github.com/ZalithLauncher/ZalithLauncher2` |
| **Core engine** | [PojavLauncher](https://github.com/PojavLauncherTeam/PojavLauncher) (JNI layer) |
| **What it does** | Android launcher for **Minecraft: Java Edition** — downloads, configures, and runs MC:JE on Android via a custom JVM + OpenGL/Vulkan translation layer |
| **License** | GPL-3.0 |
| **Package name** | `com.movtery.zalithlauncher` (suffixed `.v2` at runtime) |
| **Min SDK** | 26 (Android 8.0) |
| **Target SDK** | 35 |
| **Compile SDK** | 36 |
| **Version** | `2.4.3` (code `200025`) |
| **Language mix** | Kotlin (primary), Java (legacy/JNI bridge), C (native JNI) |
| **UI framework** | Jetpack Compose + Material Design 3 |

### Fork-specific features (vs upstream)
- Cape system for player cosmetics
- Offline account support
- Chroma (rainbow) username effects
- Home screen shortcuts
- Import/export settings and accounts (with skins & capes)
- Bug fixes over upstream

## 2. Tech Stack & Build System

### Languages & Frameworks
| Layer | Technology |
|---|---|
| UI | **Jetpack Compose** (BOM `2026.04.01`), **Material 3** (`1.5.0-alpha18`) |
| Navigation | **AndroidX Navigation3** (`1.1.1`) — uses `NavKey`-based backstack |
| DI | **Dagger Hilt** (`2.58`) |
| Networking | **Ktor** (`3.4.3`) for HTTP client/server, **OkHttp** (`5.3.2`) for downloads |
| Serialization | **Kotlinx Serialization** (JSON), **Gson** (legacy) |
| Database | **Room** (`2.8.4`) + **SQLCipher** (`4.14.1`) for encrypted storage |
| Image loading | **Coil 3** (`3.4.0`) with GIF, SVG, and Ktor network backends |
| Settings storage | **MMKV** (`1.3.14`) — Tencent's fast key-value store |
| Crash reporting | **Fishnet** (`1.1.0`) for native crash capture |
| String obfuscation | **StringFog** (XOR-based, applied to `com.movtery.zalithlauncher.info` package) |
| Native code | C via **NDK** (`25.2.9519653`), built with `ndk-build` (not CMake) |
| Native hooking | **ByteHook** (`1.0.10`) for runtime function hooking |
| JVM targets | **Java 17** (app module), **Java 11** (library modules), **Java 8** (LWJGL module) |

### Build System
- **Gradle** with Kotlin DSL (`.gradle.kts`)
- **Version catalog** at `gradle/libs.versions.toml`
- AGP version: `8.12.3`, Kotlin: `2.3.20`
- JVM args: `-Xmx4g -XX:MaxMetaspaceSize=512m`
- Pre-build task `generateInfoDistributor` auto-generates `InfoDistributor.java` with OAuth client ID, launcher name, CurseForge API key, etc. — these are read from env vars or `.txt` files in root
- Build can filter by arch via `-Darch=arm64` (options: `arm`, `arm64`, `x86`, `x86_64`, `all`)
- Assets include JRE runtimes (`jre-8`, `jre-17`, `jre-21`) which get filtered per-arch during merge

## 3. Module Architecture

The project is a multi-module Gradle build (`rootProject.name = "ZalithLauncher"`):

```
ZL2NFS/                          ← Root project
├── ZalithLauncher/              ← :ZalithLauncher — Main Android app (application module)
├── LWJGL/                       ← :LWJGL — Java library (no Android)
├── LayerController/             ← :LayerController — Android library
├── ColorPicker/                 ← :ColorPicker — Android library
└── Terracotta/                  ← :Terracotta — Android library
```

| Module | Type | Namespace | Purpose |
|---|---|---|---|
| **ZalithLauncher** | `com.android.application` | `com.movtery.zalithlauncher` | The main app. Contains all game logic, UI, native code, accounts, downloads, settings. Depends on all other modules. |
| **LWJGL** | `java-library` | `org.lwjgl.glfw` | Packages LWJGL3 GLFW classes into a JAR that gets placed into `ZalithLauncher/src/main/assets/components/lwjgl3/`. This JAR is loaded at runtime by the Minecraft JVM process. Targets Java 8. |
| **LayerController** | `com.android.library` | `com.movtery.layer_controller` | Compose-based UI library for on-screen control layout positioning, snapping, and serialization. Uses kotlinx-serialization. |
| **ColorPicker** | `com.android.library` | `com.movtery.colorpicker` | Compose-based color picker widget used in the control editor and theme settings. |
| **Terracotta** | `com.android.library` | `net.burningtnt.terracotta` | Networking library for EasyTier-based P2P multiplayer (VPN tunneling). Minimal dependencies (just AppCompat). |

### Dependency graph
```
ZalithLauncher ──depends on──▶ LayerController
                ──depends on──▶ ColorPicker
                ──depends on──▶ Terracotta
LWJGL (standalone, output JAR copied to ZalithLauncher assets)
```

## 4. Application Lifecycle & Entry Points

### Application class: `ZLApplication`
`@HiltAndroidApp` Application subclass. On `onCreate()`:
1. Sets global uncaught exception handler → writes crash to `PathManager.FILE_CRASH_REPORT`, launches `ErrorActivity`
2. Initializes **Fishnet** (native crash catcher)
3. Initializes **MMKV** and loads all settings
4. Initializes **Logger**
5. Initializes `AccountsManager` and `GamePathManager`
6. Detects device architecture (with Asus x86 workaround)
7. Configures **Coil** image loader (20MB memory cache, 512MB disk cache, GIF+SVG decoders)

### Activities (all landscape-locked)

| Activity | Role |
|---|---|
| `SplashActivity` | **Launcher entry** (`MAIN`/`LAUNCHER`). Shows splash screen, unpacks components, then navigates to `MainActivity`. Also handles modpack/controls import via `activity-alias` intents. |
| `MainActivity` | Primary launcher UI — home, account management, version management, downloads, settings. All Compose screens live here. |
| `ControlEditorActivity` | Visual editor for on-screen touch control layouts. |
| `VMActivity` | **Runs in `:game` process.** Hosts the Minecraft game surface (OpenGL/Vulkan), input handling, overlays (keyboard, menu ball, joystick, hotbar). |
| `ErrorActivity` | Displays crash logs with copy/share. |
| `FatalErrorActivity` | Dialog-style fatal error display for Application-level crashes. |

### Services

| Service | Process | Purpose |
|---|---|---|
| `GameService` | `:game` | Foreground service keeping the game process alive |
| `JvmService` | `:jvm` | Foreground service running standalone JVM tasks (e.g., Forge installer) via socket server |
| `TerracottaVPNService` | `:game` | Android VPN service for EasyTier P2P multiplayer tunneling |

### Process model
- **Main process**: Launcher UI (`SplashActivity`, `MainActivity`, `ControlEditorActivity`)
- **`:game` process**: `VMActivity` + `GameService` + `TerracottaVPNService` — isolated so game crashes don't kill the launcher
- **`:jvm` process**: `JvmService` — isolated JVM execution for mod loader installers

## 5. Package Map — `com.movtery.zalithlauncher`

```
com.movtery.zalithlauncher/
├── ZLApplication.kt              ← Application class
├── SplashException.java          ← Wrapper to distinguish splash-phase crashes
├── bridge/                       ← JNI bridge to native C code
│   ├── ZLBridge.java             ← JNI method declarations (native methods)
│   ├── ZLBridgeStates.kt         ← Shared state between Java and native (window size, renderer, etc.)
│   ├── ZLNativeInvoker.kt        ← High-level Kotlin wrappers for native calls
│   ├── LoggerBridge.java         ← Native logging bridge
│   └── NativeLibraryLoader.java  ← Loads .so libs
├── components/                   ← Asset unpacking (JRE runtimes, LWJGL components)
│   ├── jre/                      ← JRE runtime installation & management
│   ├── UnpackComponentsTask.kt   ← Unpacks bundled assets on first launch
│   └── InstallableItem.kt        ← Represents an installable component
├── context/                      ← Application context holder
├── contract/                     ← Android activity result contracts
├── coroutine/                    ← Task execution framework
│   ├── TaskSystem.kt             ← Global task queue manager
│   ├── Task.kt                   ← Base task abstraction
│   ├── TaskFlowExecutor.kt       ← Coroutine-based sequential/parallel executor
│   └── TitledTask.kt             ← User-facing task with title/progress
├── crashlogs/                    ← Crash log parsing and display
├── database/                     ← Room database
│   ├── AppDatabase.kt            ← Database definition (accounts table)
│   └── Converters.kt             ← Type converters for Room
├── game/
│   ├── account/                  ← Account management
│   │   ├── Account.kt            ← Room @Entity — the account data model
│   │   ├── AccountDao.kt         ← Room DAO for CRUD
│   │   ├── AccountType.kt        ← Enum: MICROSOFT, LOCAL
│   │   ├── AccountsManager.kt    ← Singleton managing current account
│   │   ├── AccountUtils.kt       ← Login flows, token refresh
│   │   ├── microsoft/            ← Microsoft OAuth flow
│   │   ├── offline/              ← Offline/local account creation
│   │   ├── auth_server/          ← Third-party auth server (Yggdrasil-compatible)
│   │   ├── yggdrasil/            ← Yggdrasil protocol implementation
│   │   └── wardrobe/             ← Skin & cape download/management
│   ├── addons/                   ← Mod/addon management
│   ├── control/                  ← Touch control layout data & manager
│   ├── download/
│   │   ├── assets/               ← Minecraft asset downloading (from Modrinth, CurseForge, MCIM)
│   │   │   └── platform/         ← Platform abstraction (CurseForge, Modrinth, MCIM mirrors)
│   │   ├── game/                 ← Game version downloading & installation
│   │   │   ├── GameInstaller.kt  ← Main orchestrator for downloading a MC version
│   │   │   ├── forge/            ← Forge/NeoForge installer
│   │   │   ├── fabric/           ← Fabric/Quilt installer
│   │   │   ├── optifine/         ← OptiFine installer
│   │   │   └── cleanroom/        ← Cleanroom loader installer
│   │   ├── modpack/              ← Modpack import & installation
│   │   │   └── install/          ← CurseForge/Modrinth modpack parsers & installers
│   │   └── jvm_server/           ← Isolated JVM process for running installers
│   ├── input/                    ← Keycode mapping (AWT ↔ LWJGL), character senders
│   ├── keycodes/                 ← Keycode constants
│   ├── launch/                   ← Game launch pipeline
│   │   ├── Launcher.kt           ← Builds full JVM command line
│   │   ├── LaunchArgs.kt         ← Constructs Minecraft launch arguments
│   │   ├── GameLauncher.kt       ← Orchestrates the full launch sequence
│   │   ├── JvmLauncher.kt        ← Starts the JVM process via JNI
│   │   ├── LaunchGame.kt         ← Pre-launch checks and setup
│   │   ├── MCOptions.kt          ← Parses/modifies options.txt
│   │   └── handler/              ← Launch event handlers
│   ├── multirt/                  ← Multiple Java runtime management (JRE 8/17/21)
│   ├── path/                     ← Game directory path management
│   ├── plugin/                   ← Plugin system
│   │   ├── PluginLoader.kt       ← Loads APK-based plugins
│   │   ├── renderer/             ← Renderer plugins
│   │   ├── driver/               ← Vulkan driver plugins
│   │   ├── natives/              ← Native library plugins
│   │   └── ffmpeg/               ← FFmpeg plugin integration
│   ├── renderer/                 ← Graphics renderer management
│   │   ├── Renderers.kt          ← Registry of all renderers
│   │   ├── RendererInterface.kt  ← Renderer contract
│   │   └── renderers/            ← Built-in implementations (GL4ES, Zink, VirGL, etc.)
│   ├── support/                  ← Touch controller proxy support
│   ├── text/                     ← Text/chat utilities
│   ├── version/                  ← Minecraft version JSON parsing
│   │   ├── installed/            ← Installed version management
│   │   ├── download/             ← Version manifest fetching
│   │   └── mod/, saves/, etc.    ← Per-version resource management
│   └── versioninfo/              ← Version metadata display
├── library/                      ← Open-source library credits/licensing
├── notification/                 ← Android notification helpers
├── path/                         ← Global path management
│   ├── PathManager.kt            ← All filesystem paths (dirs + files)
│   ├── UrlManager.kt             ← All URL constants + HTTP client factories
│   └── LibPath.kt                ← Native library path resolution
├── provider/                     ← Android ContentProviders (DocumentsProvider, FileProvider)
├── setting/                      ← Settings framework
│   ├── AllSettings.kt            ← All ~100+ settings defined as typed properties
│   ├── SettingsRegistry.kt       ← Base class with setting type factories
│   ├── SettingsInitializer.kt    ← Loads settings from MMKV on startup
│   ├── _MMKV.kt                  ← MMKV wrapper utilities
│   ├── enums/                    ← Setting enum types (DarkMode, Language, etc.)
│   └── unit/                     ← Setting unit types
├── terracotta/                   ← Terracotta multiplayer integration
│   ├── Terracotta.kt             ← Main controller
│   ├── TerracottaState.kt        ← Connection state machine
│   ├── TerracottaVPNService.java ← Android VPN service
│   └── profile/                  ← Network profiles
├── ui/
│   ├── activities/               ← Activity implementations (see Section 4)
│   ├── base/                     ← Base Compose components
│   ├── code_editor/              ← Code/text editor components
│   ├── components/               ← Reusable Compose UI components
│   ├── control/                  ← In-game touch controls
│   │   ├── mouse/                ← Virtual mouse cursor
│   │   ├── joystick/             ← On-screen joystick
│   │   ├── gamepad/              ← Physical gamepad support
│   │   ├── gyroscope/            ← Gyroscope-based camera control
│   │   ├── input/                ← Input event processing
│   │   └── event/                ← Control event system
│   ├── screens/                  ← All Compose screens
│   │   ├── main/                 ← MainScreen, ErrorScreen
│   │   ├── content/              ← Feature screens (AccountManage, Download, Settings, etc.)
│   │   ├── game/                 ← GameScreen (in-game overlay), JVMScreen
│   │   └── splash/               ← SplashScreen, UnpackScreen
│   ├── theme/                    ← Material 3 theming (colors, typography, palettes, festivals)
│   └── upgrade/                  ← Launcher self-update UI
├── upgrade/                      ← Launcher update checking logic
├── utils/                        ← Utility classes
│   ├── animation/                ← Transition animation types
│   ├── device/                   ← Architecture detection, Vulkan support checking
│   ├── file/                     ← File operations
│   ├── image/                    ← Image processing
│   ├── json/                     ← JSON utilities
│   ├── logging/                  ← Logger system
│   ├── network/                  ← Network utilities
│   ├── settings/                 ← Settings import/export
│   └── ...                       ← Various other utilities
└── viewmodel/                    ← All ViewModels (MVVM pattern)
    ├── AccountManageViewModel.kt ← Account CRUD, login flows
    ├── HomePageViewModel.kt      ← Home page data loading
    ├── LaunchGameViewModel.kt    ← Game launch orchestration
    ├── ModpackImportViewModel.kt ← Modpack import flow
    └── ...                       ← 16 ViewModels total
```

### Other source trees in the app module
- `org.jackhuang` — HMC library utilities (inherited from upstream)
- `org.lwjgl` — LWJGL runtime shims
- `com.oracle.dalvik` — Dalvik VM utilities

## 6. Key Subsystems Deep Dive

### 6.1 Game Launch Pipeline

The launch sequence (simplified):

1. **`LaunchGame.kt`** — Pre-launch validation: checks account login, verifies version exists, checks JRE is installed
2. **`GameLauncher.kt`** — Orchestrates the full sequence:
   - Selects Java runtime (auto-pick or user-selected from `RuntimesManager`)
   - Sets renderer via `Renderers.setCurrentRenderer()`
   - Patches `options.txt` via `MCOptions`
   - Starts `GameService` foreground service
   - Launches `VMActivity` in the `:game` process
3. **`Launcher.kt`** — Builds the complete JVM command line:
   - Classpath construction from version JSON libraries
   - JVM arguments (memory, GC, system properties)
   - Library sorting fix via `LibSortFix.kt`
4. **`LaunchArgs.kt`** — Constructs Minecraft-specific arguments:
   - Auth tokens, player name, UUID
   - Game directory, assets directory
   - Version-specific arguments from version JSON
   - Window resolution
5. **`JvmLauncher.kt`** — Actually starts the JVM via `ZLBridge` native call
6. **`VMActivity`** — Hosts the OpenGL surface, renders the game, handles all input

Key files: `game/launch/Launcher.kt` (~22KB, the big one), `game/launch/LaunchArgs.kt` (~18KB)

### 6.2 Account System

**Data model**: `Account.kt` is a Room `@Entity` with fields for tokens, profile ID, skin model type, and optional auth server URL.

**Account types** (`AccountType.kt`):
- `MICROSOFT` — OAuth device code flow via Microsoft identity
- `LOCAL` — Offline accounts (no auth, generated UUID)
- Third-party auth servers are also supported via `otherBaseUrl` field (Yggdrasil-compatible)

**Key classes**:
- `AccountsManager.kt` — Singleton. Manages current selected account, provides `StateFlow` for reactive UI updates. Backed by Room database.
- `AccountUtils.kt` — Login logic: Microsoft OAuth, token refresh, Yggdrasil login
- `wardrobe/` — Downloads skins/capes from Mojang or third-party session servers. Stores locally as PNG files keyed by account UUID.

**Storage**: Accounts in Room DB (encrypted via SQLCipher). Skins at `DIR_ACCOUNT_SKIN/<uuid>.png`, capes at `DIR_ACCOUNT_CAPE/<uuid>.png`.

### 6.3 Download & Installation Engine

#### Game versions
- `GameInstaller.kt` (~44KB) — Main orchestrator. Downloads version JSON from Mojang, resolves libraries, downloads assets, writes version to disk.
- Supports vanilla + mod loaders: **Forge**, **NeoForge**, **Fabric**, **Quilt**, **OptiFine**, **Cleanroom**
- Forge-like installers (`forge/`) run in a separate JVM process via `JvmService` + `JVMSocketServer` to avoid crashing the launcher

#### Mod/resource downloading
- Platform abstraction in `download/assets/platform/`
- Supported platforms: **Modrinth**, **CurseForge**, **MCIM** (Chinese mirror)
- `AbstractPlatformSearcher.kt` defines the search interface
- Each platform has its own subpackage with API-specific models and searchers

#### Modpack installation
- `modpack/install/ModPackInstaller.kt` — Handles both CurseForge and Modrinth modpack formats
- `ModpackImporter.kt` — Imports `.mrpack` / `.zip` modpack files
- Downloads are managed through the `TaskSystem` with progress tracking

#### Mirror sources
- Configurable via `AllSettings.fileDownloadSource`, `assetSearchSource`, etc.
- Options: `OFFICIAL_FIRST`, `MIRROR_FIRST`, `OFFICIAL_ONLY`, `MIRROR_ONLY` (enum `MirrorSourceType`)

### 6.4 Renderer System

`Renderers.kt` is a singleton registry. Built-in renderers:

| Renderer | ID contains | Description |
|---|---|---|
| `NGGL4ESRenderer` | `gl4es` | Next-gen GL4ES (OpenGL → OpenGL ES translation) |
| `GL4ESRenderer` | `gl4es` | Original GL4ES |
| `VulkanZinkRenderer` | `vulkan`, `zink` | Zink (OpenGL → Vulkan translation) |
| `VirGLRenderer` | `virgl` | VirGL (GPU virtualization) |
| `FreedrenoRenderer` | `freedreno` | Freedreno (Qualcomm Adreno, direct) |
| `PanfrostRenderer` | `panfrost` | Panfrost (ARM Mali, direct) |

**Compatibility filtering**: Renderers containing `vulkan` are hidden if device lacks Vulkan. Renderers containing `zink` are hidden on 32-bit x86.

**Plugin renderers**: Additional renderers can be loaded from APK plugins via `plugin/renderer/`. These are discovered at runtime and added to the registry via `Renderers.addRenderer()`.

**Vulkan drivers**: The `AllSettings.vulkanDriver` setting selects which Turnip driver to use. Driver plugins are loaded from `plugin/driver/`.

### 6.5 Input & Control System

This is one of the most complex parts. It translates Android touch/gamepad/gyroscope input into Minecraft-compatible keyboard/mouse events.

**Layers**:
1. **Touch controls** (`ui/control/`) — Virtual on-screen buttons defined by JSON layout files stored in `DIR_CONTROL_LAYOUTS`. Edited via `ControlEditorActivity`.
2. **Virtual mouse** (`ui/control/mouse/`) — Configurable cursor with custom hotspots per cursor type (arrow, crosshair, I-beam, etc.). Modes: slide, click.
3. **Keyboard** (`ui/control/Keyboard.kt`, ~27KB) — On-screen keyboard rendering and key event dispatch.
4. **Joystick** (`ui/control/joystick/`) — On-screen analog stick for WASD movement, with dead zone, lock-forward, and sprint support.
5. **Gamepad** (`ui/control/gamepad/`) — Physical gamepad mapping with remappable buttons. Dual joystick: one for movement, one for camera.
6. **Gyroscope** (`ui/control/gyroscope/`) — Gyroscope-based camera control with sensitivity, smoothing, axis inversion.
7. **Hotbar** (`ui/control/Hotbar.kt`) — Touch-based hotbar slot selection.

**Input bridge**: `game/input/` contains `AWTCharSender`, `LWJGLCharSender`, and `EfficientAndroidLWJGLKeycode` which translate between Android keycodes, AWT keycodes, and LWJGL keycodes.

**Native bridge**: Input events ultimately go through `ZLBridge` → `input_bridge_v3.c` (JNI) → the running JVM's LWJGL input system.

### 6.6 Settings Architecture

**Pattern**: `AllSettings` is a singleton `object` extending `SettingsRegistry`. Each setting is a typed property created via factory methods:
- `boolSetting(key, default)` → `SettingUnit<Boolean>`
- `intSetting(key, default, range)` → `SettingUnit<Int>`
- `stringSetting(key, default)` → `SettingUnit<String>`
- `enumSetting(key, default)` → `SettingUnit<E>`
- `longSetting`, `offsetSetting`, `parcelableSetting`, `stringListSetting`

**Storage backend**: MMKV (memory-mapped key-value, by Tencent). Very fast, crash-safe.

**Reading**: `AllSettings.renderer.getValue()` or similar.
**Writing**: `AllSettings.renderer.setValue(newValue)`.

**Categories** (~100+ settings total):
- **Renderer**: resolution, VSync, Zink driver, surface view, performance mode, shader dump
- **Game**: version isolation, JVM args, RAM allocation, Java runtime, log settings
- **Controls**: mouse sensitivity, gamepad mapping, gyroscope, gesture controls
- **Launcher**: theme, dark mode, language, animations, background, home page type
- **Multiplayer**: Terracotta enable/config
- **Special styles**: joystick control placement and behavior

**Import/export**: Settings can be exported to / imported from JSON files via `utils/settings/`.

### 6.7 Plugin System

`game/plugin/PluginLoader.kt` discovers and loads APK-based plugins from the device.

**Plugin types**:
- **Renderer plugins** (`plugin/renderer/`) — Additional OpenGL/Vulkan renderers (e.g., FCL Renderer Plugin)
- **Driver plugins** (`plugin/driver/`) — Custom Vulkan drivers (e.g., Turnip builds)
- **Native library plugins** (`plugin/natives/`) — Additional native libs injected at launch
- **FFmpeg plugins** (`plugin/ffmpeg/`) — FFmpeg binary for video recording

**How it works**: Plugins are regular APKs installed on the device. `ApkPluginManager.kt` scans for them, `PluginLoader.kt` extracts their native libraries and metadata. Plugin icons are cached in `_PluginIconCache.kt`.

**Settings**: `AllSettings.disableNativeLibPlugins` stores a list of disabled plugin identifiers.

### 6.8 Terracotta (Multiplayer / VPN)

Terracotta enables P2P multiplayer by creating a VPN tunnel between players using [EasyTier](https://easytier.cn/).

**Components**:
- `Terracotta.kt` — Main controller, manages EasyTier process lifecycle
- `TerracottaState.kt` — Connection state machine (disconnected → connecting → connected)
- `TerracottaVPNService.java` — Android `VpnService` implementation that routes game traffic through the EasyTier tunnel
- `TerracottaNodeList.kt` — Manages connected peer list
- `profile/` — Network profile configuration (IP, port, network name)

**How it works**: The Terracotta module (`:Terracotta` library) provides the core networking. The app module integrates it with an Android VPN service running in the `:game` process, so game traffic is transparently routed through the P2P tunnel.

**Config**: Enabled via `AllSettings.enableTerracotta`. Logs written to `FILE_TERRACOTTA_LOG`.

### 6.9 Task / Coroutine System

`coroutine/TaskSystem.kt` is a global singleton managing all background tasks (downloads, installations, unpacking, etc.).

**Key classes**:
- `Task.kt` — Base abstraction with `execute()` suspend function, progress tracking, cancellation
- `TitledTask.kt` — Extends `Task` with user-visible title and description
- `TaskFlowExecutor.kt` — Runs tasks sequentially or in parallel using Kotlin coroutines
- `TaskState.kt` — Enum: `PENDING`, `RUNNING`, `COMPLETED`, `FAILED`, `CANCELLED`
- `DataBridge.kt` — Bridge for passing data between tasks
- `MutableTransitionStateFlow.kt` — Custom `StateFlow` with transition animations

**Usage pattern**:
```kotlin
TaskSystem.submitTask(TitledTask("Downloading...") {
    // suspend code here
    updateProgress(50)
})
```

The UI observes task state via `FlowExtensions.kt` which provides reactive collection utilities.

## 7. Native Layer (JNI)

Native code lives in `ZalithLauncher/src/main/jni/`. Build system: `ndk-build` via `Android.mk`.

| File | Purpose |
|---|---|
| `jre_launcher.c` | Launches the embedded JRE, sets up environment, calls `JNI_CreateJavaVM` |
| `input_bridge_v3.c` (~28KB) | **Core input bridge** — receives mouse/keyboard/touch events from Java and injects them into the LWJGL event queue inside the running JVM |
| `egl_bridge.c` | EGL context creation, surface management, renderer selection (GL4ES, Zink, VirGL, etc.) |
| `awt_bridge.c` | AWT window system bridge for headless Java rendering |
| `awt_xawt/` | X11 AWT shims |
| `exit_hook.c` | Hooks `exit()` to prevent abrupt process termination |
| `java_exec_hooks.c` | Hooks `exec` family to intercept Java subprocess creation |
| `lwjgl_dlopen_hook.c` | Hooks `dlopen` to redirect LWJGL native library loading |
| `stdio_is.c` | Redirects stdout/stderr to Android logcat |
| `bigcoreaffinity.c` | Pins threads to big CPU cores for performance |
| `utils.c` | Shared utility functions |
| `ctxbridges/` | Context bridge implementations for different rendering backends |
| `driver_helper/` | Vulkan driver loading helpers |
| `environ/` | Environment variable setup |
| `linkerhook/` | Dynamic linker hooks (for library path manipulation) |
| `logger/` | Native logging system |
| `GL/` | OpenGL header shims |

**Key native libraries loaded**: `libpojavexec.so` (main bridge), plus renderer-specific `.so` files.

## 8. UI Architecture

### Framework
Full **Jetpack Compose** with **Material 3**. No XML layouts. Navigation uses **AndroidX Navigation3** with custom `NavKey` types.

### Navigation model
Defined in `ui/screens/`:
- `NormalNavKey.kt` — Top-level navigation keys (Home, Download, Settings, etc.)
- `NestedNavKey.kt` — Nested navigation within screens
- `BackStackNavKey.kt` — Backstack-aware navigation keys
- `TitledNavKey.kt` — Keys that carry a display title
- `_Navigation.kt` — Navigation setup and transitions

### Screen hierarchy
```
SplashActivity
  └── SplashScreen → UnpackScreen → (launches MainActivity)

MainActivity
  └── MainScreen (top-level scaffold with navigation rail/bar)
      ├── LauncherScreen (home page, version select, play button)
      ├── AccountManageScreen
      ├── VersionsManageScreen
      ├── DownloadScreen
      ├── SettingsScreen
      ├── MultiplayerScreen
      └── ...nested screens (VersionSettings, VersionExport, FileSelector, etc.)

VMActivity (separate process)
  └── GameScreen (OpenGL surface + overlays)
      ├── Keyboard overlay
      ├── Mouse cursor
      ├── Joystick
      ├── Hotbar
      ├── Menu ball (floating action button)
      └── JVMScreen (for Forge installer progress)
```

### Theming
`ui/theme/Theme.kt` (~38KB) — Extensive Material 3 theme with:
- Multiple built-in color themes (`ColorThemeType` enum)
- Dynamic color support (Android 12+)
- Custom color option
- Dark mode (system, light, dark)
- Festival-specific effects (`feativals/` — typo in codebase, means "festivals")
- Custom typography via `Type.kt`

### MVVM Pattern
All state management uses `ViewModel` + `StateFlow`. 16 ViewModels in `viewmodel/` package. Key ones:
- `AccountManageViewModel` (~39KB) — Largest VM, handles all account operations
- `HomePageViewModel` — Home page data, news feed
- `LaunchGameViewModel` — Game launch state machine
- `LauncherUpgradeViewModel` — Self-update checking
- `ModpackImportViewModel` — Modpack import flow

## 9. Path & URL Management

### Filesystem Paths (`PathManager.kt`)
All paths are initialized in `PathManager.refreshPaths(context)`. Key directories:

| Path constant | Location | Purpose |
|---|---|---|
| `DIR_GAME` | `filesDir/games` | Root game directory |
| `DIR_MULTIRT` | `filesDir/games/runtimes` | JRE installations (8, 17, 21) |
| `DIR_COMPONENTS` | `filesDir/components` | Unpacked LWJGL, etc. |
| `DIR_CONTROL_LAYOUTS` | `externalFiles/control_layouts` | Custom touch control JSON files |
| `DIR_ACCOUNT_SKIN` | `filesDir/games/account_skins` | Downloaded skin PNGs |
| `DIR_ACCOUNT_CAPE` | `filesDir/games/account_capes` | Downloaded cape PNGs |
| `DIR_BACKGROUND` | `filesDir/background` | Custom launcher backgrounds |
| `DIR_LAUNCHER_LOGS` | `externalFiles/logs` | Launcher log files |
| `DIR_NATIVE_LOGS` | `externalFiles/logs/native` | Native crash logs (Fishnet) |
| `DIR_TERRACOTTA` | `filesDir/net.burningtnt.terracotta` | Terracotta config/data |
| `DIR_CACHE` | `cacheDir` | Temp files (downloads, modpacks) |

### URL Constants (`UrlManager.kt`)
All API endpoints defined as `const val`:
- `URL_MINECRAFT_VERSION_REPOS` — Mojang version manifest
- `URL_PROJECT_RELEASES_LATEST` — GitHub API for self-update
- `URL_GITHUB_RENDERER_PLUGINS` — Plugin download links
- HTTP clients: `GLOBAL_CLIENT` (Ktor), `createOkHttpClient()` (OkHttp)
- User-Agent: `ZL2+/Android_<version>`

## 10. Dependency Injection (Hilt)

- `ZLApplication` is annotated `@HiltAndroidApp`
- Hilt is used for injecting dependencies into Activities and ViewModels
- KSP is used for Hilt annotation processing (not kapt)
- Most singletons in the codebase are `object` singletons (Kotlin), not Hilt `@Singleton`s — this is a pattern choice, not all DI goes through Hilt

## 11. Database (Room + SQLCipher)

- `AppDatabase.kt` defines the Room database with one entity: `Account`
- Database is encrypted using **SQLCipher** (`net.zetetic:sqlcipher-android`)
- `Converters.kt` provides type converters for custom types stored in Room
- DAO: `AccountDao.kt` — standard CRUD operations
- Database files stored in `DIR_DATA_BASES` (parent of `filesDir`)

## 12. Key Configuration Files

| File | Purpose |
|---|---|
| `gradle.properties` (root) | JVM args, AndroidX config |
| `ZalithLauncher/gradle.properties` | Launcher name (`Diavlon Launcher XL`), version (`2.4.3`/`200025`), OAuth/CurseForge API keys, signing passwords |
| `gradle/libs.versions.toml` | Version catalog — all dependency versions |
| `settings.gradle.kts` | Module includes |
| `.oauth_client_id.txt` | (gitignored) Microsoft OAuth client ID |
| `.curseforge_api.txt` | (gitignored) CurseForge API key |
| `.store_password.txt` / `.key_password.txt` | (gitignored) Release keystore passwords |
| `ZalithLauncher/src/main/jni/Android.mk` | NDK build configuration |
| `ZalithLauncher/src/main/jni/Application.mk` | NDK application config (ABI, STL) |
| `ZalithLauncher/src/main/AndroidManifest.xml` | Android manifest (activities, services, permissions, providers) |

## 13. Build Variants & Signing

### Build types
- **release**: No minification, no shrinking, signed with `debugBuild` config (note: release uses debug signing by default for easy distribution)
- **debug**: `applicationIdSuffix = ".debug"`, `versionNameSuffix = "-debug"`, signed with `debugBuild` config

### Signing configs
- **releaseBuild**: `zalith_launcher.jks` — passwords from env vars (`STORE_PASSWORD`, `KEY_PASSWORD`) or `.txt` files
- **debugBuild**: `zalith_launcher_debug.jks` — hardcoded default passwords in `gradle.properties`

### APK naming
Format: `DiavlonLauncher XL-<version>.apk` (or with ABI suffix if split builds enabled)

### ABI splits
Optional, controlled by `-Darch=<arm|arm64|x86|x86_64>`. Default: `all` (universal APK).

## 14. Common Pitfalls & Gotchas

1. **Multi-process architecture**: `VMActivity` and `GameService` run in `:game` process, `JvmService` in `:jvm` process. Singletons and static state are **not shared** across processes. Communication happens via intents, AIDL, or socket (`JVMSocketServer`).

2. **StringFog obfuscation**: The `com.movtery.zalithlauncher.info` package is XOR-obfuscated at build time. The auto-generated `InfoDistributor.java` contains sensitive API keys — never log its constants in plaintext.

3. **Generated source**: `InfoDistributor.java` is generated by the `generateInfoDistributor` Gradle task into `build/generated/source/zalith/java`. Don't edit it manually — it's rebuilt every build.

4. **JRE assets are huge**: The app bundles JRE 8, 17, and 21 runtimes as compressed tar.xz files in `assets/runtimes/`. This makes the APK very large. The arch-filtering in the build script (`-Darch=arm64`) is essential for CI to produce smaller APKs.

5. **LWJGL JAR output path**: The `:LWJGL` module outputs its JAR directly to `ZalithLauncher/src/main/assets/components/lwjgl3/`. This is a build artifact that lives in the source tree — don't delete it.

6. **Native build**: Uses `ndk-build` (not CMake). The `Android.mk` file is at `src/main/jni/Android.mk`. NDK version must be exactly `25.2.9519653`.

7. **Room + SQLCipher**: The database is encrypted. You can't inspect it with standard SQLite tools. Use SQLCipher CLI or the app's own export functionality.

8. **Chinese comments**: Many code comments are in Chinese (Simplified). This is inherited from the original Zalith Launcher codebase. The translation is: comments describe the function/purpose of the code.

9. **Settings are MMKV-backed, not SharedPreferences**: Don't use Android's `SharedPreferences` API. Always go through `AllSettings.*` properties.

10. **Compose Navigation3**: This is a newer navigation library (not the classic `navigation-compose`). It uses `NavKey` objects instead of string routes. Check `_Navigation.kt` and the `*NavKey.kt` files for patterns.

---

*Last updated: 2026-05-16. Generated from source analysis of the `master` branch.*
