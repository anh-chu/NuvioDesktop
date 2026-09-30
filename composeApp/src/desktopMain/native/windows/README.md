# Windows player bridge

`player_bridge.cpp` compiles to `player_bridge.dll`, the JNI layer between Nuvio's Kotlin desktop
player and `libmpv`. It also owns the WebView2 on-screen controls.

## Building

```powershell
.\gradlew.bat :composeApp:buildWindowsPlayerBridge
```

Output: `composeApp\build\native\windows\player_bridge.dll`.

Requirements:

- An MSVC toolchain. The task shells out to `vswhere.exe` and `vcvars64.bat`, and fails with
  `Visual Studio C++ toolchain was not found` when they are missing.
- The `Microsoft.Web.WebView2` NuGet package under
  `%USERPROFILE%\.nuget\packages\microsoft.web.webview2`, or pass
  `-Pnuvio.webview2.dir=C:\path\to\microsoft.web.webview2\<version>`. The task needs `WebView2.h`
  and `WebView2Loader.dll.lib`.
- A JDK — JNI headers are taken from Gradle's `java.home`.

> **The task only runs when its output does not already exist.**
>
> `buildWindowsPlayerBridge` is guarded by
> `onlyIf { !windowsPlayerBridgeOutput.get().asFile.exists() }`. After editing the source, delete
> `composeApp\build\native\windows\player_bridge.dll` first or the build silently does nothing.

### Building without a local Windows toolchain

`.github/workflows/build-windows-player-bridge.yml` runs the same task on a hosted `windows-latest`
runner, for contributors who have no MSVC or WebView2 SDK installed:

1. Fork `NuvioMedia/NuvioDesktop` and enable Actions in the fork's **Actions** tab — forks ship
   with Actions disabled.
2. Get `.github/workflows/build-windows-player-bridge.yml` onto the fork's **default branch
   (`Dev`)**. GitHub shows the *Run workflow* button only for workflows that exist on the default
   branch, and the API/CLI can only dispatch a workflow that has already run at least once — so a
   file that lives only on your feature branch is not dispatchable. Committing **just this file**
   to `Dev` is enough; `ref` (next step) selects the code that actually gets built, so your C++
   changes can stay on your feature branch.
3. **Actions → Build Windows player bridge → Run workflow**, with `ref` set to that branch.
4. Download the `player_bridge` artifact. The run summary prints the DLL's size and SHA-256.

Once the PR is merged upstream the file is on `Dev` for everyone and steps 1–2 are unnecessary.

The workflow checks out the revision, installs a JDK, extracts the `Microsoft.Web.WebView2` NuGet
package (version pinned by the `webview2_version` input) into the runner's temp directory, passes it
as `-Pnuvio.webview2.dir`, runs `:composeApp:buildWindowsPlayerBridge`, and uploads
`player_bridge.dll`, `player_bridge.lib` and `player_bridge.pdb`.

It builds the bridge and nothing else — no assembly, packaging, signing or release, and no
repository secrets. Git LFS is not fetched, so the ~115 MB bundled libmpv runtime is skipped; this
task does not read it. `gradle.properties` asks for `-Xmx12288M`, which rules out the smaller
private-repo runners.

### Where the DLL is loaded from

`NativePlayerBridge.loadNativeLibrary()` tries, in order:

1. `<compose.application.resources.dir>\native\windows\player_bridge.dll` — the packaged install
   location.
2. `composeApp\build\native\windows\player_bridge.dll` — a local build, when the working directory
   is the repo root.
3. `/native/windows/player_bridge.dll` on the classpath, extracted to
   `%LOCALAPPDATA%\Nuvio\Cache\native-player-bridge\windows\<content-hash>\`.

Installed builds use (3): the DLL ships inside `composeApp-desktop-*.jar`. The cache directory is
named after a SHA-256 over the file names *and* their bytes, so replacing the jar entry creates a
new cache directory on the next launch — no manual cache clearing is needed.

To use (1) instead, create `<resources dir>\native\windows\` and copy in `player_bridge.dll` **plus**
`WebView2Loader.dll`, `vcruntime140*.dll`, `msvcp140*.dll`, `concrt140.dll` and `libmpv-2.dll`.
Those are resolved relative to the bridge, and a missing `WebView2Loader.dll` makes `System.load`
fail.

> The DLL is JNI-coupled to `NativePlayerBridge.kt`. Build it from the source revision matching the
> installed app; a mismatch surfaces as `UnsatisfiedLinkError` or a crash.

## Environment variables (advanced, unsupported)

Both are read when playback starts, so Nuvio has to be restarted with them already present in the
launching process.

| Variable | Effect |
| --- | --- |
| `NUVIO_LIBMPV_PATH` | Absolute path to a replacement `libmpv-2.dll`. Tried before the bundled copy and loaded with `LOAD_WITH_ALTERED_SEARCH_PATH`, so DLLs sitting next to it are found first. |
| `NUVIO_MPV_CONFIG_DIR` | Absolute path to an mpv configuration directory. When set, the bridge passes `config-dir` and `config=yes` to libmpv, so `mpv.conf`, `input.conf`, `scripts/` and `shaders/` are read from there. When unset, config loading stays disabled and playback behaviour is unchanged. |

Example with a standalone AnimeJaNai install:

```powershell
$env:NUVIO_LIBMPV_PATH = "C:\AnimeJaNai\libmpv-2.dll"
$env:NUVIO_MPV_CONFIG_DIR = "C:\AnimeJaNai\portable_config"
Start-Process "C:\Program Files\Nuvio\Nuvio.exe"
```

Use `[Environment]::SetEnvironmentVariable(...)` to persist them.

Diagnostics go to the debugger output stream (`OutputDebugStringW`, prefix `[NuvioPlayerBridge]`)
and are visible in DebugView or a debugger. Nothing is logged when neither variable is set.

### Option precedence

mpv parses the configuration directory inside `mpv_initialize()`, which runs **after** the bridge
has applied Nuvio's defaults, and `mp_parse_cfgfiles()` passes no `M_SETOPT_PRESERVE_CMDLINE` or
`M_SETOPT_NO_OVERWRITE` flags. Therefore **every option your `mpv.conf` names overrides the
corresponding Nuvio default.** That is the point of an opt-in hatch, but it also means a config can
change options Nuvio itself relies on.

With a stock AnimeJaNai `portable_config`:

| Your config sets | Overrides Nuvio's | Consequence |
| --- | --- | --- |
| `hwdec=nvdec` | `hwdec=auto` | Intended. |
| `vo=gpu-next` | `vo=gpu-next` | No change. |
| `gpu-api=vulkan,auto` | `gpu-api=auto` / `d3d11` | Nuvio embeds mpv in its own child window; Vulkan is the least tested path here. |
| `profile=high-quality` | `scale=spline36` | Heavier scalers, higher GPU cost. |
| `vf=@aji:animejanai:…` | `vf=d3d11vpp=…` (RTX Video Super Resolution) | The two filters are mutually exclusive; RTX VSR silently turns off. |
| `volume-max=100` | `volume-max=200` | Volume above 100% is clamped. |
| `cache-secs=30` | `cache-secs=36000` | The buffered-range indicator stops showing ahead-buffer past ~30 s. |
| `save-position-on-quit=yes` | *(unset)* | mpv writes its own watch-later files into the config directory and may compete with Nuvio's resume handling. |
| `alang=jpn`, `slang=eng` | *(unset)* | Affects mpv's automatic track pick; Nuvio still re-selects explicitly after load. |

If playback breaks, unset both variables and restart Nuvio to return to stock behaviour.

### Known limitations

- **Keybindings never reach mpv.** Nuvio handles keyboard input in its own control layer and passes
  `input-vo-keyboard=no`. mpv also only auto-loads `input.conf`, not AnimeJaNai's
  `input-animejanai.conf`. So AnimeJaNai's slot-switch bindings (`!`, `@`, `SHARP`, `Ctrl+1`–
  `Ctrl+9`) and its `Ctrl+j` stats binding are inactive. Lua scripts under `scripts/` do load.
- There is no settings UI, directory picker or per-option editor; this is deliberately
  environment-variable only.
- A `NUVIO_MPV_CONFIG_DIR` that does not point at an existing directory fails playback with a
  visible error rather than silently falling back to stock behaviour.
- Putting `wid=` in your own `mpv.conf` will break embedding: the bridge sets `wid` before
  `mpv_initialize()`, and the config file is read during it.
