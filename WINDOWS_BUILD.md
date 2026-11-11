# Building Eglantine on Windows (MSVC) — from a clean machine

Written by a new AMD tester. It covers everything I had to install and run, starting from a Windows 11 PC that only had Git and Python.
Branch: `nx2-experimental` @ `e03bcb4a` (2026-11-11). It mirrors the `msvc` job in `.github/workflows/build-windows.yml`.

> Status: **tested end to end** on Windows 11 (Ryzen 5 7500F, 6 cores, 16 GB RAM, RX 7900 GRE) with VS 2022 Build Tools (MSVC 19.44), CMake 4.4.4, Ninja 1.13.2, Vulkan SDK 1.4.341.1 and Qt 6.8.3. The resulting `eglantine.exe` starts and shows `Eglantine | e03bcb4aea-nx2-experimental | MSVC 19.44.35230.0`.
>
> Timings: first configure ~10 min (mostly CPM downloads). Configure with the CPM cache already filled: under 1 min. Full Release build of `yuzu` + `yuzu-cmd`: ~14 min.

---

## 1. Requirements at a glance

| Tool | Version used | Required? | Install command |
|---|---|---|---|
| Git for Windows | any recent | yes | `winget install Git.Git` |
| Python | 3.10+ (3.14 works) | yes (CPM / externals) | `winget install Python.Python.3.13` |
| Visual Studio 2022 **Build Tools** (or Community) with *Desktop development with C++* | MSVC 14.44 | yes | see §2 |
| CMake | 3.31+ (4.4.4 used) | yes | `winget install Kitware.CMake` |
| Ninja | 1.13.2 | recommended (CI uses it) | `winget install Ninja-build.Ninja` |
| Vulkan SDK | **1.4.341.1** (the version the repo script installs; docs say ≥ 1.3.274) | yes | see §3 |
| Qt 6 (msvc2022_64 + qtcharts) | 6.8.3 (CI version) | **optional**: bundled by default under MSVC | see §4 |

Disk: plan for about 15 GB (Build Tools ~6–8 GB, Vulkan SDK ~1 GB, Qt ~1.5 GB, build tree + CPM cache several GB).

## 2. Visual Studio Build Tools (no full IDE needed)

The full IDE is not required. The Build Tools are enough:

```powershell
winget install --id Microsoft.VisualStudio.2022.BuildTools -e --override "--wait --passive --norestart --add Microsoft.VisualStudio.Workload.VCTools --includeRecommended --add Microsoft.VisualStudio.Component.VC.Tools.x86.x64 --add Microsoft.VisualStudio.Component.Windows11SDK.26100"
```

Asks for UAC elevation. Restart the terminal afterwards so `PATH` updates.

**Gotcha: several VS versions installed.** `tools/windows/load-msvc-env.ps1` runs `vswhere -latest`. If VS 2026 ("18") is also installed, it picks VS 2026, not 2022. The docs conflict on this point: `/Build` says VS 2022 / `Visual Studio 17 2022`, while `/Deps` mentions `Visual Studio 17 2026` (which is not a valid generator name; VS 2026 is `Visual Studio 18 2026`). The docs should state which version is supported and how to force one (e.g. `vswhere -version "[17.0,18.0)"`).

## 3. Vulkan SDK

`tools/windows/install-vulkan-sdk.ps1` works, but it **must run from an elevated PowerShell**. The docs don't say this. It also **does not set `VULKAN_SDK` or `PATH` outside GitHub Actions**: it only writes `GITHUB_ENV`/`GITHUB_PATH`. On a dev machine you have to set them yourself:

On Windows PowerShell 5.1, the script's `WebClient` / `Invoke-WebRequest` download of the ~300 MB installer can be extremely slow and look stuck. It stalled for me. Downloading with `curl.exe -L -o ...` (built into Windows) and running the installer with the same arguments is much faster:
`vulkansdk-windows-X64-1.4.341.1.exe --root "C:/VulkanSDK/1.4.341.1" --accept-licenses --default-answer --confirm-command install`

```powershell
# elevated PowerShell, from the repo root
./tools/windows/install-vulkan-sdk.ps1
[Environment]::SetEnvironmentVariable('VULKAN_SDK', 'C:\VulkanSDK\1.4.341.1', 'User')
# then add C:\VulkanSDK\1.4.341.1\Bin to your user PATH and open a new terminal
glslangValidator --version   # must work before configuring
```

## 4. Qt — the bundled Qt does NOT link with VS 2022

`CMakeLists.txt:80`: `YUZU_USE_BUNDLED_QT` **defaults to ON when using MSVC**, so CMake downloads a prebuilt Qt by itself (no Qt account needed). The docs don't mention this.

**But with VS 2022 that default route fails at the final link of `eglantine.exe`.** Steps to reproduce: no Qt flags, MSVC 14.44 (VS 2022 Build Tools 17.14). CPM fetches a **static Qt 6.11.1** (`.cache/cpm/qt6/6.11.1-windows-amd64`). Configure and compile then succeed (`eden-cli.exe` links fine), but `eglantine.exe` fails with:

```
Qt6Core.lib(qbytearray.cpp.obj) : error LNK2019: unresolved external symbol __std_replace_copy_1 ...
bin\eglantine.exe : fatal error LNK1120: 6 unresolved externals
```

Missing symbols: `__std_replace_copy_1/2`, `__std_rotate`, `__std_unique_4/8`, `__std_minmax_element_2u`, referenced from `Qt6Core.lib` / `Qt6Gui.lib`. These are vectorized STL helpers that only exist in newer MSVC STLs. The bundled static Qt was compiled with a newer toolset (VS 2026 / MSVC 14.5x), so it can't be linked with VS 2022's STL. This is probably why `/Deps` mentions VS 2026.

So you have two working options:
- **A. VS 2026** (MSVC 14.5x) with the default bundled Qt. I didn't test it here.
- **B. VS 2022 + your own Qt 6.8.3** (what CI does, and what I verified). Pass `-DYUZU_USE_BUNDLED_QT=OFF -DCMAKE_PREFIX_PATH=...`. **Use a fresh build directory if you switch.** Deleting `CMakeCache.txt` does fix `Qt6_DIR`, but the build then fails with `fatal error C1189: "This file was generated using the moc from 6.11.1..."`, because QuaZip's autogen moc files are stale. Start from an empty build dir. `.cache/cpm` can stay.

Suggestion for the project: pin a bundled Qt built with VS 2022, or make CMake fail early with a clear message when the MSVC version is too old for the bundled Qt.

The configure warning `Could NOT find Qt6Test` / `Failed to find optional Qt component "Test"` (from QuaZip) is harmless.

Install Qt 6.8.3 without a Qt account using aqtinstall (~1.5 GB, a few minutes):

```powershell
python -m pip install --user aqtinstall
python -m aqt install-qt windows desktop 6.8.3 win64_msvc2022_64 -m qtcharts -O C:/Qt
# then pass -DCMAKE_PREFIX_PATH=C:/Qt/6.8.3/msvc2022_64 at configure time
```

## 5. Clone

```powershell
git clone --branch nx2-experimental https://github.com/Eglantine-Community/Eglantine.git
cd Eglantine
```

Note: the `/Build` page still clones `github.com/Djihads80/Eglantine.git`. It redirects, but it should point to `Eglantine-Community/Eglantine`.

## 6. Load the MSVC environment, configure, build

Run everything in **one** PowerShell session (the env script only affects the current process):

```powershell
. ./tools/windows/load-msvc-env.ps1     # or open "x64 Native Tools Command Prompt for VS 2022"

cmake -S . -B build/release -G Ninja `
  -DCMAKE_BUILD_TYPE=Release `
  -DBUILD_TESTING=OFF -DYUZU_TESTS=OFF `
  -DYUZU_CMD=ON `
  -DYUZU_ROOM=OFF -DYUZU_ROOM_STANDALONE=OFF `
  -DYUZU_USE_BUNDLED_SDL3=ON `
  -DENABLE_UPDATE_CHECKER=OFF `
  -DYUZU_USE_BUNDLED_QT=OFF `
  -DCMAKE_PREFIX_PATH=C:/Qt/6.8.3/msvc2022_64     # drop these two lines on VS 2026 (option A)

cmake --build build/release --target yuzu yuzu-cmd --parallel
C:/Qt/6.8.3/msvc2022_64/bin/windeployqt.exe --release --no-translations build/release/bin/eglantine.exe
```

Output: `build/release/bin/eglantine.exe` (Qt UI) and `build/release/bin/eden-cli.exe`. `windeployqt` copies the Qt DLLs next to the exe so it can run.

Useful variants:
- `-DCMAKE_BUILD_TYPE=RelWithDebInfo` gives usable stack traces while testing crashes.
- If you use a compiler cache (sccache/ccache) with Debug builds, set `CL=/FS`.

## 7. Troubleshooting

- Configure fails on dependencies: delete `.cache/cpm` and `build/release/CMakeCache.txt`, then reconfigure.
- `glslangValidator` not found: `VULKAN_SDK`/`PATH` were not set (see §3), or the terminal is older than the install.
- Wrong VS picked: see §2.
- `LNK2019 __std_replace_copy_1` / `__std_rotate` / `__std_unique_*` in `Qt6Core.lib`: you are on VS 2022 with the bundled Qt (see §4).
- `C1189 "This file was generated using the moc from 6.x"`: Qt version changed in an existing build dir. Use a fresh build dir.

## 8. Suggested doc fixes (summary)

1. Update the clone URL to `Eglantine-Community/Eglantine`.
2. **Bundled Qt (the MSVC default) fails to link with VS 2022** (§4). Either require VS 2026 for the bundled route, or document `-DYUZU_USE_BUNDLED_QT=OFF` + Qt 6.8.3 through aqtinstall (no Qt account needed). Better still, ship a bundled Qt built with VS 2022, or add a CMake check that fails early.
3. `install-vulkan-sdk.ps1` needs admin rights and does not set `VULKAN_SDK`/`PATH` locally (or fix the script to do it).
4. Settle the VS 2022 vs 2026 question, and fix the invalid generator name `Visual Studio 17 2026` on `/Deps`.
5. Document the Build Tools-only route (one winget command) as the lightest option.
6. Show the full CI-equivalent configure line plus the targets (`yuzu`, `yuzu-cmd`), and where the binaries end up.
7. Add a "prebuilt binaries" pointer: GitHub Releases has daily builds and per-branch pre-releases, so testers don't have to compile.
