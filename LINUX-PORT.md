# Linux port

This branch carries the changes that make [YuE2-Studio](https://github.com/timoncool/YuE2-Studio)
run on Linux. Upstream targets Windows only; nothing here is an official port.

Base: **v3.2.0**. Built and verified on an Intel Arc B580 through Vulkan, where a full song
generates end to end: `[Pipeline] Done: 1 tracks, 162.8 s of audio in 142.5 s (1.1x realtime)`
with `[Load] LM/NAR/VAE backend: Vulkan0`.

## The commits

Each is a `#[cfg]`-gated patch in upstream's own style, so Windows behaviour is untouched.

| Commit | Why |
|---|---|
| `hardware: name a Linux adapter…` | `display_adapter()` returned a literal `None` off Windows, so `gpu_name` was always empty. That is not cosmetic: `device_chain` only offers Vulkan when a card was found, so the default **Auto** backend fell straight to the processor. Now reads DRM sysfs and names the card through `lspci -nn`. |
| `device chain: ask the one place that knows…` | The Vulkan entry tested `hardware.gpu_name.is_some()`, the same condition `uses_vulkan()` already expresses — use the helper so the two cannot drift. |
| `engine runtime: the Visual C++ runtime is a Windows requirement` | `vc_runtime_missing()` demanded `vcruntime140.dll` and friends unconditionally, refusing to start a complete engine with *"the engine bundle is incomplete"*. |
| `process: bind children to the studio's lifetime off Windows too` | Windows gets this from a job object; `bind_children_to_this_process` and `adopt` are both no-ops off Windows, so a hard-killed studio left `yue-server` holding the GPU and port 18087. The next start then failed with `cannot bind 127.0.0.1:18087`, which reads like a config problem. Now `PR_SET_PDEATHSIG(SIGKILL)` in the forked child, plus a `getppid()` re-check for the fork window. SIGKILL because the engine ignores SIGTERM mid-inference. |
| `setup: do not claim a download that is not happening` | The runtime status object's mere presence selected a "Downloading the engine libraries — 0%" step. On Vulkan nothing is ever downloaded, so that is what a start screen showed while the engine was loading. |
| `lockfiles: the Linux resolutions…` | Keeps the Tauri-generated Linux schema beside the tracked Windows one. |

## Building

```bash
source ../.tools/project-env.sh          # CARGO_HOME/RUSTUP_HOME off the read-only $HOME

# engine: the Vulkan backend, not CUDA
cd ../yue2cpp && ./buildvulkan.sh        # needs vulkan-headers spirv-headers glslc
# do NOT use buildsycl.sh - there are no SYCL sources in this tree

cd ../yue2-studio
cargo build --release -p music-server
npm --prefix app install && npm --prefix app run build
npm --prefix desktop install && cd desktop && npm run build
```

Run it with `YUE_ENGINE_BIN` pointing at the built `yue-server`, `YUE_MODELS_ROOT` at the
weights, and `LD_LIBRARY_PATH` including the engine's build directory — the engine bakes an
absolute `RUNPATH` at link time, so a moved tree will not find `libggml.so.0`.

## Known gaps

- **`cargo test -p music-server` cannot link on Linux, and this is an upstream bug, not from
  this branch.** Pristine v3.2.0 fails identically, before any patch here:
  `rust-lld: error: unable to find library -lkernel32`. `cargo build`, `cargo check
  --tests` and the release binaries are all fine; only the test binary's link fails.
- **Stem separation does not work.** `lyrics_sync.rs` locates the ONNX runtime by the
  hardcoded name `onnxruntime.dll`, and downloads `onnxruntime-win-x64-*.zip`. On Linux the
  runtime directory ends up holding headers and no library, so separation reports no runtime.
  A native `libonnxruntime.so` is not looked for. Not yet fixed.
- **The UI device picker offers only Auto/CUDA/CPU** though `ComputeBackend` supports Vulkan.
  Auto works correctly with these patches, so it is cosmetic.
- **`totalVramGb` reads 0.** The Intel `xe` driver exposes no VRAM total in sysfs and
  `vulkaninfo --json` needs device access. Only affects the suggested model profile.

## Engine pin

`engines/yue2-cpp-source.json` pins `fc8f97bf`, one commit on from the `6585c96c` this branch
was verified against — a sibling on the same `adapters` branch, accepting the same arguments.
Rebuild the engine at the pin if you want it exactly.
