# Eglantine on AMD: findings and patches

What an AMD tester found running `nx2-experimental` with Mario Kart World on an RX 7900 GRE, the patches written for it, how each one measured, and where 3-4 player split screen hits a wall in the GPU thread split.

| | |
|---|---|
| Base | `nx2-experimental` @ `e03bcb4a` (2026-11-11) |
| GPU | RX 7900 GRE, Adrenalin 32.0.32015.7029 |
| CPU | Ryzen 5 7500F (6C/12T), 16 GB |
| Build | MSVC 19.44 (VS 2022), Release ([build notes](WINDOWS_BUILD.md)) |
| Title | `0400C3F00006E000`, 30 fps mode, FSR3 frame gen |

## Summary

| # | Change | Effect measured | Status |
|---|---|---|---|
| 1 | Keep Eden's VkPipelineCache on AMD | Boot replay of known pipelines: ~117 s → ~4 s | tested |
| 2 | Frame gen pacer: no 1 s stabilize on an isolated hitch | Frame gen off 34% → 9% of a race; output 38.5 → 56.2 fps | tested |
| 3 | Async shaders on Switch 2: skip new pipelines for redrawn screen-sized targets | Solo pipeline waits: 132 (8.3 s) → 9 (0.4 s) | tested; a regression was fixed, the fix is untested |
| 4 | Pass met ≥ acquires without draining the split's back | ~1,800-2,300 drains/s avoided, no fps gain | works, no net gain |
| 5 | Pool deleted SparseTiles images (aliased mode) | Targets eight 0.5-1.2 s freezes per 9 min session | untested |

Each change is its own branch off `nx2-experimental` on this fork, with the measurements in the commit message. Nothing was pushed or proposed to the main repository.

## Changes

### 1. Eden's pipeline cache is needed on AMD — tested

Branch [`fix/amd-vk-pipeline-cache`](https://github.com/riyadhHDA/Eglantine/tree/fix/amd-vk-pipeline-cache) · `16b011391b` · `vk_pipeline_cache.cpp`, 1 line + comment

- **Problem:** `70b39fe1d2` left the pipeline cache to NVIDIA's and AMD's drivers. Its own numbers (2.26 vs 2.46 ms per pipeline) are NVIDIA's driver cache. On AMD's Windows driver, pipelines replayed from the disk shader cache still took ~110 ms each: 1,388 pipelines kept the two replay workers busy for ~117 s after every boot.
- **Change:** `DriverCachesPipelines` returns true for `VK_DRIVER_ID_NVIDIA_PROPRIETARY` only.
- **Result:** second boot, 2,162 pipelines, slow builds (> 50 ms) stop after ~4 s instead of ~117 s. `vulkan_pipelines.mythos.bin` is 1.1 MB for this title (the original commit's concern was a 245 MB file).

### 2. Frame generation dropped out on every hitch — tested

Branch [`fix/frame-gen-isolated-hitch`](https://github.com/riyadhHDA/Eglantine/tree/fix/frame-gen-isolated-hitch) · `559ae0f7b3` · `frame_gen_pacer.cpp/.h`

- **Problem:** any base interval over 100 ms (`MINIMUM_BASE_RATE`) called `Stabilize()`: no generation for a full second. With hitches every few seconds the output alternated between ~60 and ~25 fps, which felt worse than frame gen off.
- **Change:** a hitch under 500 ms with no other hitch in the previous second only skips generation for that frame and leaves `smoothed_interval` alone. Pauses (≥ 500 ms) and repeated hitches still stabilize.
- **Result:** race with FSR3 on: generation off 34% → 9% of seconds, output 38.5 → 56.2 fps, seconds at ≥ 55 fps 22% → 81%. This build also had change 1, which reduces hitches on its own.

### 3. Async shaders were off for every Switch 2 draw — regression fixed, fix untested

Branch [`fix/nx2-async-redrawn-targets`](https://github.com/riyadhHDA/Eglantine/tree/fix/nx2-async-redrawn-targets) · `d351ef7145`, `1f0ac17fdb`, `d848bc68cc` · `vk_pipeline_cache.cpp`, `texture_cache.h`, `image_base.h`

- **Problem:** `b7980efd7f` (MKW grass) returns early in `BuiltPipeline` when `native_turing` is set, which `Nx2MacroContext()` sets for every Switch 2 channel. So every new pipeline is waited on. With ~100 ms AMD compiles, 85% of submission waits (603, up to 636 ms) were ordinary depth-tested scene draws.
- **Change:** new `TextureCache::DrawTargetRedrawnEachFrame()`: the draw's color target 0 (or its depth buffer when it has none) is at least 640×360 and was written during one of the two frames before this one. `ImageBase` gains `previous_written_frame`. Depth-tested native draws into such a target are skipped while their pipeline builds (`miss=async-build-redrawn`); everything else still waits.
- **Result:** solo race, 132 waits (8.3 s) → 9 (0.4 s). No missing grass reported.
- **Caveat:** the version without the size limit garbled the text of an end-of-race dialog: small 128×128 targets drawn once per frame on consecutive frames passed as "redrawn". The 640×360 floor (`d848bc68cc`) excludes them; that version is untested. The two-frame window exists because the frame counter can step twice between two guest frames at 30 fps.

### 4. Semaphore acquires drained the back — works, no net gain

Branch [`perf/nx2-acquire-early-met`](https://github.com/riyadhHDA/Eglantine/tree/perf/nx2-acquire-early-met) · `113f203007` · `puller.cpp`, `nx2_perf.h`, `nx2_profile.h`

- **Problem:** every Ampere semaphore trigger acquire, and every poll of a pending one, calls `DrainBack` before reading the word. In 3-player races that was ~280 ms/s of front waits (`acquire trigger`).
- **Change:** a greater-or-equal acquire that guest memory already meets returns before the drain: these semaphores only grow, so a newer value still pending in the back or the buffer cache can only be larger. Nx2Perf switch `acquire_early_met`; under `EDEN_NX2_PERF_VERIFY=1` the drain still runs and disagreements are logged (none seen). Profiler line: `acquires met early`.
- **Result:** 1,700-2,300 acquires/s met early, acquire-trigger waits 281 → ~190 ms/s, but `list writers` waits rose ~200 → ~330 ms/s and fps did not move. The back is the limit (see below). Not worth proposing on its own.

### 5. ~1 s freezes recreating huge SparseTiles images — untested

Branch [`perf/amd-sparse-tiles-pool`](https://github.com/riyadhHDA/Eglantine/tree/perf/amd-sparse-tiles-pool) · `87851e53b4` · `vk_texture_cache.cpp/.h`

- **Problem:** on AMD, SparseTiles images use the aliased mode: `BindZeroBlock` binds every block to one zeroed block on the graphics queue, then `fence.Wait()`. For 16384×16384 (256×256 blocks) that is ~5,500 binds and ~1 s. MKW deletes and recreates the same four images (2× 16384², 2× 8192²) each time it remaps its sparse regions (`StartTiles: the game maps its tiles`). Eight freezes of 0.5-1.2 s in a 9-minute session lined up with those recreations.
- **Change:** a deleted aliased SparseTiles image goes to a pool (8 entries) with its `SparseLayerMemory` and VkImage, memory declared first so it is released after the image. The same image (address, format, size, levels) created again takes it back and runs `ClearTiles`, with no bind. Images whose zeroed block was dropped (`BackSparseTilesWhole`) are not pooled. Logged as `reused with its bindings`.
- **Open:** not run yet. A semaphore instead of the CPU fence wait would avoid the stall in general, but the scheduler's wait mask has no transfer stage, which the clear needs.

## 3-4 player split screen: bound by the back thread

Split screen is unplayable on this machine (7-10 fps in race). It is not the host GPU: the front's GPU fence wait stays ~4 ms per frame in every run. It is the number of draws the split's back can record per second.

| Race | Draws / frame | Draws / s (back) | Real fps |
|---|---|---|---|
| Solo | ~1,500-3,000 | 41,000-60,000 | 22-30 |
| 3 players, session A | 5,400-9,400 | 55,000-77,000 | 8-10 |
| 3 players, session B (with change 4) | 8,200-11,300 | 65,000-75,000 | ~7 |

> **The ceiling is ~70,000 draws/s, about 13-14 µs of back time per draw.** With split screen the title submits 3-4× the draws per frame, so fps drops by the same factor. Reaching solo frame rates needs the back to be ~3× faster per draw, or draw recording spread over more than one thread.

### Where the time goes (`EDEN_NX2_GPU_PROFILE`, ms per second)

| Zone | Solo | 3 players | Note |
|---|---|---|---|
| Front: push (command parsing) | 434 | 400-580 | front ~90% busy in 3P |
| Front: split drain (waits for back) | 190 | 185-300 | ~250 drains per frame in 3P vs ~107 solo |
| · of which acquire trigger | 164 | 281 → 190* | *with change 4 |
| · of which list writers | ~200-300 | 200 → 330* | `dma_pusher.cpp:224`, every GP entry |
| Back: method + back replay | ~293 | ~317-400 | ring dispatch and register writes |
| Back: split wait (waits for front) | 276 | 112-210 | both threads wait on each other |
| Back: draw setup zones (update buffers, bind uniform stream, fill image views, render targets, pipeline lookup, …) | ~200 | ~240-300 | per-draw cost roughly flat |

### Suggestions for whoever picks this up

- **List writers drain.** `DmaPusher` calls `WaitForWriters()` before every entry not yet known as GPU-written, to catch command lists that a queued dispatch writes for the first time (the Fast Fusion device-loss case). Once acquire drains are gone this is the dominant front wait. Knowing which pages a queued dispatch can write (output buffers of the culling pass) would let the front wait only when the entry is on one. Not touched here without a way to test the device-loss case.
- **Per-draw back cost.** `NX2_GPU_THREAD_SPLIT.md` already says the next step is the back's draw setup. Split screen is the best stress case for it: ~10,000 draws per frame with mostly repeated state across the three views.
- **Multi-threaded recording.** The three views render into separate viewports of the same targets. Recording per-view secondary command buffers on workers would be the structural fix, and a large one.
- Parsec was used for the extra players and adds its own capture/encode load on a 6-core CPU. It does not explain a 3× drop.

## Other notes

- **Windows build docs.** With VS 2022, the default bundled Qt (static 6.11.1) fails to link `eglantine.exe` (`LNK2019 __std_replace_copy_1`, `__std_rotate`, `__std_unique_4/8` in Qt6Core/Qt6Gui): it was built with a newer MSVC STL. `-DYUZU_USE_BUNDLED_QT=OFF` with Qt 6.8.3 from aqtinstall (as CI does) works. Switching Qt in an existing build dir then fails with a stale QuaZip moc (`C1189`). The Vulkan SDK script needs admin rights and doesn't set `VULKAN_SDK` outside Actions. Full notes: [WINDOWS_BUILD.md](WINDOWS_BUILD.md).
- The `Draw skipped` and `ReportPipelineBuildWait` logs are capped (64 and 400 per process), so long sessions under-report late skips.
- One 191 ms wait on a new compute pipeline remained in solo; compute dispatches were left alone.

---

Tested by an AMD tester on their own machine. Analysis and patches done with Claude Opus 5.5 (log and profile analysis, code changes); commits carry an `Assisted-by` footer per the project's AI policy. Logs from every session are available on request.
