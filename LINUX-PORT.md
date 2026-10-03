# Linux port — and how to carry it to the next upstream release

This branch makes [YuE2-Studio](https://github.com/timoncool/YuE2-Studio) run on Linux.
Upstream targets Windows only; nothing here is an official port.

**Base: v3.4.0.** Verified end to end on an Intel Arc B580 through Vulkan — generated
`152.4 s of audio in 160.5 s (0.9x realtime)` with `[studio] computing on Vulkan` and the
decoder companion loaded (`Companion …: 396 decoder keys`).

If you are an agent picking this up: read **Part 3** before touching anything, and **Part 4**
before deciding you are done. The traps in Part 4 have each cost hours and are not
discoverable by reading the code.

---

## Part 1 — Run it

```bash
source ../.tools/project-env.sh          # CARGO_HOME/RUSTUP_HOME off the read-only $HOME

cd ../yue2cpp && ./buildvulkan.sh        # ~10 min; the GPU path
cd ../yue2-studio
cargo build --release -p music-server
npm --prefix app install && npm --prefix app run build
npm --prefix desktop install && (cd desktop && npm run build)
```

Then launch through the workspace's `run-studio.sh`, which supplies the three paths a
`tauri build --no-bundle` layout cannot bundle:

| Variable | Why |
|---|---|
| `YUE_ENGINE_BIN` | else it looks in `<exe dir>/resources/yue2-cpp/`, which `--no-bundle` never creates |
| `YUE_MODELS_ROOT` | the weights; otherwise it defaults to `~/.local/share/yue2-studio/models/yue2-cpp` |
| `LD_LIBRARY_PATH` | **load-bearing** — see trap 2 |
| `YUE_STUDIO_DATA_ROOT` | library, settings, logs |

Requirements beyond a normal Rust/Node box: `vulkan-headers`, `spirv-headers`, `glslc`,
`glslangValidator` (all in the Arch `extra` repo).

Model set: `YuE2-3B-Q8_0.gguf`, `YuE2-Vae-F32.gguf`, `SheetSage2-Q8_0.gguf`, and — **required
from 3.4** — `nar_lora_joint_v9.safetensors` (140 MB, `Mothersuperior/yue2-mothersuperior-realaudio-tokenizer-v4`,
revision `e2e63d859f3af879baf1b4d4e9f22d1eeda6fde5`). Every profile includes the companion, so
without it no profile is complete and the engine will not start.

---

## Part 2 — The patch series

Six commits, each `#[cfg]`-gated in upstream's own style so **Windows behaviour is unchanged**.
Upstream has fixed none of these, verified against v3.4.0 source.

1. **`hardware: name a Linux adapter`** — `display_adapter()` returned a literal `None` off
   Windows, so `gpu_name` was always empty. Not cosmetic: `device_chain` only offers Vulkan
   when a card was found, so the default **Auto** backend fell straight to the processor on a
   machine whose GPU works. Now reads DRM sysfs (`/sys/class/drm/card*/device/{vendor,device}`)
   and names the card through `lspci -nn`, falling back to a vendor label.
2. **`device chain: ask the one place that knows`** — the Vulkan entry tested
   `hardware.gpu_name.is_some()` directly, which is the condition `uses_vulkan()` already
   expresses. Route through the helper so the two cannot drift apart.
3. **`engine runtime: the Visual C++ runtime is a Windows requirement`** —
   `vc_runtime_missing()` demanded `vcruntime140.dll` et al unconditionally and refused to
   start a complete engine with *"the engine bundle is incomplete"*.
4. **`process: bind children to the studio's lifetime off Windows too`** — Windows gets this
   from a job object; `bind_children_to_this_process` and `adopt` are no-ops off Windows, so a
   hard-killed studio left `yue-server` holding the GPU and port 18087, and the next start
   failed with `cannot bind 127.0.0.1:18087` — which reads like a config problem. Now
   `PR_SET_PDEATHSIG(SIGKILL)` in the forked child, plus a `getppid()` re-check for the fork
   window. SIGKILL because the engine ignores SIGTERM mid-inference.
5. **`setup: do not claim a download that is not happening`** — the runtime status object's
   *presence* selected a "Downloading the engine libraries — 0%" step. On Vulkan nothing is
   ever downloaded, so that is what a start screen showed while the engine was loading.
6. **`lockfiles`** — regenerated for Linux, plus the Tauri-generated Linux schema beside the
   tracked Windows one.

---

## Part 3 — Porting to the next upstream release

This took ~40 minutes on the 3.2.0 → 3.4.0 jump. Order matters.

```bash
cd repos/yue2-studio
git fetch --tags origin
git switch -c linux-port-<ver> v<ver>          # always base on the tag

# 1. Re-apply the six patches. Expect clean or near-clean.
git cherry-pick <the six SHAs from the previous port branch>

# 2. Find your new port branch's SHAs for next time
git log --oneline v<ver>..HEAD
```

Before cherry-picking, **check which patched files upstream touched** — it tells you where the
conflicts will be:

```bash
git diff --numstat v<old> v<new> -- crates/music-server/src/hardware.rs \
    crates/music-server/src/lib.rs crates/music-server/src/engine_runtime.rs \
    crates/music-server/src/assistant_runtime.rs crates/music-core/src/process.rs \
    crates/music-engine/src/yue_server.rs app/components/EngineStarting.tsx
```

On 3.2.0 → 3.4.0 four of those files were **untouched**, so only `lib.rs` (+831 −53) and
`yue_server.rs` (+10 −2) needed care — and even those applied without conflict. Re-applying by
hand is only needed when a cherry-pick actually conflicts.

Then, in this order:

```bash
# 3. Engine: build the commit the NEW release pins, never the repo default.
cat repos/yue2-studio/engines/yue2-cpp-source.json     # read the pin
cd repos/yue2cpp
git fetch --depth 1 origin <pin> && git checkout <pin>
git submodule update --init --recursive                 # the pin may want a NEWER ggml
rm -rf build && ./buildvulkan.sh

# 4. New required model files, if the release added any (check model_manager.rs
#    `fn profiles`). 3.4 added the companion; every profile needs it.

# 5. Build the service, then the app, then verify (Part 4).
```

**Do not skip the engine rebuild** even when the old engine appears to work. The pin moves for
a reason, and an engine missing a newly-passed flag dies with
`[Server] FATAL: unknown argument --<flag>`.

---

## Part 4 — Verifying, and the traps

### The verification that actually proves it

A build succeeding proves almost nothing here. Check each of these:

```bash
# 1. GPU is seen (our hardware probe — must not be null)
curl -s localhost:8791/setup/status | python3 -c "import json,sys;print(json.load(sys.stdin)['hardware']['gpuName'])"
#    expect: Intel Corporation Battlemage G21

# 2. Engine is on the GPU, not the processor
grep -E 'backend|computing on' studio-data/engine.log | tail -3
#    expect: [Load] KV backend: Vulkan0  /  [studio] computing on Vulkan

# 3. A real generation completes
curl -s -X POST -H 'Content-Type: application/json' \
  -d '{"style":"solo piano","lyrics":"[instrumental]","duration":15,"cot":"off","steps":8,"seed":1}' \
  localhost:8791/v1/music/jobs
grep 'Pipeline\] Done' studio-data/engine.log | tail -1
```

`[studio] computing on the processor` is the failure that matters — it means the GPU was not
found, and it is *silent*: everything still works, just 400× slower.

### Trap 1 — the engine commit, and the trap in `git clone`

`git clone --depth 1` of `timoncool/yue2.cpp` lands on the **default branch**, which on this
repo is a *different, older line* than the `adapters` branch the studio pins. That build does
not accept `--adapters` and the engine dies instantly:

```
[Server] FATAL: unknown argument --adapters
```

Always build the commit in `engines/yue2-cpp-source.json`, and always
`git submodule update --init --recursive` — the pin routinely wants a different `ggml`.
This one cost an hour and looked like a dozen other things first.

### Trap 2 — the engine's RUNPATH is absolute

```
$ readelf -d repos/yue2cpp/build/yue-server | grep -i runpath
Library runpath: [/home/skai/Projects/dsh-hole/yue2cpp/build]   # the OLD location
```

Move the tree and the engine cannot find `libggml.so.0`. Nothing in the studio sets a library
path for the engine on Linux — it assumes the Windows DLL search. `run-studio.sh` exports
`LD_LIBRARY_PATH`, which is the only thing making it work. To make it robust instead, relink
with `-DCMAKE_BUILD_RPATH='$ORIGIN'` (`patchelf` was not available to patch it in place).

### Trap 3 — a stale binary looks exactly like a broken fix

**The service is compiled *into* the Tauri shell.** Editing `crates/music-server` changes
nothing until the app is rebuilt, and the running GUI then behaves like the old code with no
sign anything is out of date. This cost a whole debugging session: the GUI sat on
"Downloading the engine libraries — 0%" forever because a pre-fix binary still reported no
GPU. `run-studio.sh` now warns when any `.rs` is newer than the built app.

Rebuild **both**: `cargo build -p music-server` is only for headless `--service` mode.

### Trap 4 — orphans, and how they corrupt your testing

A studio killed rather than closed leaves `yue-server` holding the GPU and port 18087, and the
next start fails with `cannot bind`. Worse for debugging: **a stale process on 8791 answers
your requests and makes a freshly built binary look like it works — or look broken.** A
`/health` reply reported `service_executable: "...(deleted)"` while the new binary had failed
to bind. Always check first:

```bash
ps -eo pid,comm | grep -E 'yue-server|music-server'
ss -ltn | grep -E '8791|18087'
```

The fix in Part 2 commit 4 stops new orphans; it does not clean up old ones.

### Trap 5 — killing the engine demotes Vulkan for the session

An out-of-band kill of the engine makes the supervisor log
`Vulkan failed on this machine; leaving it for this session` and pin the processor. The state
is **in-memory only**, so restarting the studio clears it. Never SIGKILL the engine while
debugging unless you intend to restart the GUI after.

### Trap 6 — `cargo test` cannot link, and it is not your fault

```
rust-lld: error: unable to find library -lkernel32
```

**Pristine upstream fails identically** — verified in a clean git worktree with zero patches.
It affects the *test binary* link only; `cargo build`, `cargo check --tests` and the release
binaries are all fine. Do not go hunting through this port's patches for it. If you want to
prove that to yourself, the cheap way is a worktree and `cargo test --no-run`.

### Sandbox traps (DSH harness)

- **Do not use `nohup`/`&`/`disown` for long work.** Each `bash` call is an isolated sandbox and
  the process is killed when the call returns — a half-finished build then looks like a silent
  stall. Use the harness's background jobs, with output to a file you can read later. `nohup`
  *appeared* to survive once, which makes it worse: never trust it.
- `/tmp` is **not** shared between calls. A file written in one call is gone in the next.
- `$HOME` is read-only → `CARGO_HOME`, `RUSTUP_HOME`, `npm_config_cache` must point into
  `.tools/`.
- `ps`/`/proc` show only the current call's own namespace, so you cannot inspect another call's
  children. Watch log files and artifact counts instead.
- `/dev/dri` is hidden: anything touching the GPU (`vulkaninfo`, a real engine start) needs
  `sandbox_permissions: danger-full-access`.
- A repo pinning its own `rust-toolchain.toml` channel (this one pins
  `stable-x86_64-pc-windows-msvc`) makes rustup fetch the wrong toolchain — override with
  `RUSTUP_TOOLCHAIN=stable-x86_64-unknown-linux-gnu` rather than editing the repo.

### Build caches are path-dependent

After moving the tree, `mp3lame-sys`'s autotools step kept absolute paths from the old
location and failed with `No rule to make target '.../old/path/.../common.c'`. `cargo clean`
in the affected workspace (the main one **and** `desktop/src-tauri`, which is separate) is the
fix. Budget for a full rebuild after any move.

---

## Part 5 — Known gaps

- **Stem separation does not work.** `lyrics_sync.rs` locates the ONNX runtime by the
  hardcoded name `onnxruntime.dll` and downloads `onnxruntime-win-x64-*.zip`. On Linux the
  runtime directory ends up holding headers and no library, so separation reports no runtime.
  A native `libonnxruntime.so` is never looked for — and one is usually already installed.
  This is the highest-value next fix. The pattern to follow is commit 3: a `#[cfg]` split that
  keeps the `.dll` path on Windows and returns the `.so` elsewhere.
- **Training is CUDA-13-only by upstream design** (3.4 gates it on driver 580+). Not offered
  on this hardware; not a bug in the port.
- **`cargo test -p music-server` cannot link** — upstream regression, see trap 6.
- **`totalVramGb` reads 0.** The Intel `xe` driver exposes no VRAM total in sysfs and
  `vulkaninfo --json` needs device access. `nvtop -s` does report it as JSON
  (`mem_total: 12809404416` = 11.93 GiB) via DRM ioctls. Only affects the suggested profile.
  Note 11.93 GiB is *below* the `>= 12.0` threshold, so even a correct probe recommends
  `quality-q8`, not `native`.
- **The UI device picker offers only Auto/CUDA/CPU** though `ComputeBackend` supports Vulkan.
  Cosmetic — Auto works correctly with these patches.
- **Windows-only release scripts** (`scripts/build-*.ps1`, NSIS) were not ported. Building is
  the manual sequence in Part 1.

## Part 6 — Measured behaviour, so you do not re-derive it

- **VRAM is not the constraint.** A 130 s song peaks at **4.66 GiB of 11.93 GiB** with the GPU
  at **97–99%**. The engine evicts every model between stages (`[Store] Unload NAR`), so it
  holds one stage's weights at a time; `--keep-loaded` changes that. For comparison, the same
  card running `audio.cpp` on the same task peaked at **6.57 GiB**.
- **NAR is the whole cost.** On a 130 s song at the studio default `steps: 32`:
  AR `29 s`, NAR `355 s` (89%), VAE `1.8 s`. NAR's per-step cost scales with the latent
  sequence, which is as long as the song, so `steps × frames` compounds. At `steps: 8` the
  same work is ~15× faster. `duration` is only advisory — a 135 s request produced 128 s.
- **Lowering `steps` is a real quality tradeoff**, not free: measured SNR against the
  reference render puts 32→8 (17 dB) and 32→16 (21 dB) as *larger* departures than 32→48
  (28 dB).
- **The engine pin's `--companion` is new in 3.4.** It merges a LoRA into every NAR render;
  expect NAR to be somewhat slower than on 3.2.0 (14.6 s/step vs ~12 s/step observed here).
  Worth measuring properly rather than assuming.
