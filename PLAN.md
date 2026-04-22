# godot-vlc — GPU output (D3D11) plan

Target: **Godot 4.3+**, **libvlc 4.0 nightly**, **Windows x64**, **D3D11 → D3D12 zero-copy**.

Sibling project `../libvlc-gdextension/` is a parallel C++ GDExtension pursuing the same zero-copy path; this plan deliberately mirrors its architecture where it makes sense and diverges where godot-vlc's existing Rust surface demands it.

## Goals

- Add a GPU output backend: VLC decodes + renders into a D3D11 shared texture; Godot's D3D12 backend imports it zero-copy via `texture_create_from_extension` and samples it through a `Texture2Drd`.
- Keep the current software path intact. It remains the default backend. GPU is opt-in per player via a new `force_hardware` property.
- Preserve `VLCMediaPlayer`'s public surface (properties, signals, `get_texture() -> Texture2D`). Script callers should not need changes; only the concrete texture subclass behind `get_texture()` swaps (`ImageTexture` → `Texture2Drd`) when GPU is active.
- Audio pipeline untouched — godot-vlc already routes audio through Godot's `AudioStream`; that continues to work for both backends.

## Non-goals (v1)

- Linux / macOS GPU paths. Vulkan external-memory interop on Linux is a v1.5 item; macOS stays on the CPU path.
- HDR / 10-bit passthrough. Force `DXGI_FORMAT_R8G8B8A8_UNORM` so VLC inserts its own tone/colour conversion pass.
- Godot's Vulkan backend on Windows. Users who want the GPU path must launch with `--rendering-driver d3d12`; otherwise runtime falls back to the CPU path with a clear log message.
- Replacing or touching the audio pipeline.

## Backend selection (runtime)

Two new switches:

- **Build-time**: cargo feature `gpu`. Enabled by default on Windows (`cfg(windows)`), disabled elsewhere. Gates the `windows` / `windows-sys` dependency and the entire new `gpu_d3d11` module. When `gpu` is off the binary is byte-identical to today's.
- **Runtime**: `VLCMediaPlayer.force_hardware: bool`, `#[export]`, default `false`. When `true` the player attempts the D3D11 path; on any prerequisite failure (D3D12 driver not active, LUID match fails, `D3D11CreateDevice` errors, `texture_create_from_extension` rejects the handle) it logs an error via `godot_error!` and falls back to the software path for that player. The fallback is per-player, not global.

`force_hardware = false` (the default) is a hard guarantee of today's behaviour — no new code runs in that path.

## Binding confirmations (godot-rust 0.3.5, `api-4-3`)

All load-bearing Godot APIs exist at the crate/feature pin already declared in `Cargo.toml`:

- `RenderingDevice::texture_create_from_extension(TextureType, DataFormat, TextureSamples, TextureUsageBits, image: u64, width: u64, height: u64, depth: u64, layers: u64) -> Rid`
- `RenderingDevice::get_driver_resource(...)` with `DriverResource::LOGICAL_DEVICE` → returns `ID3D12Device*` as `u64` on the D3D12 backend.
- `Texture2Drd::set_texture_rd_rid(Rid)`.
- `RenderingServer::call_on_render_thread(Callable)`.
- Typed signal accessor: `RenderingServer::signals().frame_pre_draw()`.
- `Engine::get_frames_drawn() -> i32` for the deferred-delete queue.

No `api-4-x` bump needed.

## Repo layout changes

```
godot-vlc/
  Cargo.toml                           # + windows crate (behind `gpu` feature), default-features on Windows
  build.rs                             # unchanged — bindgen picks up libvlc_video_set_output_callbacks automatically
  src/
    lib.rs                             # unchanged module tree
    vlc_media_player.rs                # refactor: backend-selector in init()/register_player_callbacks()
    vlc_media_player/
      internal_audio_stream.rs         # unchanged
      internal_audio_stream_playback.rs# unchanged
      software_video.rs                # NEW — today's RV24 callbacks lifted out of vlc_media_player.rs verbatim
      gpu_d3d11/                       # NEW — only compiled with feature = "gpu"
        mod.rs                         # backend entry point + prerequisite checks
        adapter.rs                     # LUID lookup (Godot D3D12 device → matching DXGI adapter → D3D11 device)
        output_callbacks.rs            # C-ABI setup/cleanup/update_output/swap callbacks for libvlc
        shared_texture.rs              # ID3D11Texture2D + RTV + shared NT handle + ID3D11Fence
        rd_import.rs                   # ID3D12Device::OpenSharedHandle → texture_create_from_extension → Rid
        frame_sync.rs                  # frame_pre_draw handler enqueuing ID3D12CommandQueue::Wait
        event_queue.rs                 # SPSC queue (VLC render thread → importer)
  PLAN.md                              # this file
```

The existing CPU implementation in `src/vlc_media_player.rs:834-919` gets moved verbatim into `src/vlc_media_player/software_video.rs` in Phase 0 — pure extract-function refactor, no behaviour change. All subsequent phases only add to the GPU module; they never touch the software path.

## Architecture (GPU path)

1. **Backend pick** (in `VLCMediaPlayer::init` / `on_notification(READY)`): if feature `gpu` is compiled in **and** `force_hardware == true`, try `gpu_d3d11::try_init(&self)`. Success → use `Texture2Drd` + `libvlc_video_set_output_callbacks(libvlc_video_engine_d3d11, ...)`. Failure → log + fall through to existing `software_video` path.
2. **Adapter match** (`adapter.rs`):
   - `RenderingServer::get_rendering_device()` → `RenderingDevice::get_driver_resource(LOGICAL_DEVICE, Rid::invalid(), 0)` → `u64` → `ID3D12Device*`.
   - `ID3D12Device::GetAdapterLuid()`.
   - `CreateDXGIFactory1` → walk adapters → match LUID → `IDXGIAdapter1*`.
   - `D3D11CreateDevice(adapter, D3D_DRIVER_TYPE_UNKNOWN, ...)` with `D3D11_CREATE_DEVICE_BGRA_SUPPORT | D3D11_CREATE_DEVICE_VIDEO_SUPPORT` (latter required for hardware decode).
   - Factored behind an `AdapterEnumerator` trait so LUID matching is unit-testable without a live DXGI factory.
3. **VLC output callbacks** (`output_callbacks.rs`) — all C-ABI, all `unsafe extern "C"`:
   - `setup_cb`: hand VLC the D3D11 device + immediate context (+ an optional context mutex; null for now since we don't touch the context outside callbacks).
   - `update_output_cb`: create an `ID3D11Texture2D` with `BindFlags = RENDER_TARGET | SHADER_RESOURCE`, `MiscFlags = SHARED_NTHANDLE | SHARED_KEYEDMUTEX`, format `DXGI_FORMAT_R8G8B8A8_UNORM`; create RTV; `IDXGIResource1::CreateSharedHandle`; push `{handle, width, height}` to the SPSC queue. Must not touch Godot objects.
   - `swap_cb`: signal the shared `ID3D11Fence` with a monotonically incrementing value; store as `AtomicU64`.
   - `cleanup_cb`: release RTV, texture, D3D11 device.
   - Unused for the D3D11 engine: `makeCurrent_cb`, `getProcAddress_cb`, `metadata_cb`, `select_plane_cb` — all pass `None`. `window_cb` unused in v1.
4. **Importer** (`rd_import.rs`) — runs on a dedicated thread (not Godot main, not VLC render):
   - Drains the SPSC queue. For each `{handle, w, h}`: `ID3D12Device::OpenSharedHandle(handle, IID_PPV_ARGS(&res))` → `ID3D12Resource*`.
   - `RenderingDevice::texture_create_from_extension(TEXTURE_TYPE_2D, DATA_FORMAT_R8G8B8A8_UNORM, TEXTURE_SAMPLES_1, TEXTURE_USAGE_SAMPLING_BIT, res as u64, w, h, 1, 1)` → `Rid`.
   - Assign `Texture2Drd.set_texture_rd_rid(rid)`. The setter internally routes to `call_on_render_thread`, so we don't marshal threads ourselves.
   - Put the old `Rid` on a deferred-delete queue keyed by `Engine::get_frames_drawn()`; free after advance of ≥ `FRAMES_IN_FLIGHT` (use 3).
5. **Frame sync** (`frame_sync.rs`):
   - Connect `RenderingServer::signals().frame_pre_draw()` once at GPU-backend init.
   - On fire: read `latest_signalled` atomic, enqueue `ID3D12CommandQueue::Wait(fence, latest_signalled)` on Godot's direct queue (retrieved via `get_driver_resource(COMMAND_QUEUE, Rid::invalid(), 0)` — will confirm in Phase 2 this returns a usable queue pointer).

Threading: godot-vlc already enables `experimental-threads` on the `godot` crate, so the importer thread is fine. The VLC render thread must never touch `Gd<T>` — all marshalling is via the SPSC queue + atomics.

## Phases

Each phase lands with verification. Unit tests use Rust's built-in `#[cfg(test)]` harness for pure helpers (LUID matching, state transitions). Integration tests go in `demo/tests/phaseN_*.gd` run headlessly via `"$GODOT_BIN" --headless --path demo --script res://tests/<case>.gd`; scripts extend `SceneTree` and call `quit(exit_code)`. D3D12 debug layer is enabled for all integration runs; any validation warning fails the test.

### Phase 0 — Refactor: extract software path, add feature flag

No behavioural change. Pure scaffolding.

- Move today's `video_lock_callback` / `video_unlock_callback` / `video_display_callback` / `video_format_callback` / `video_cleanup_callback` out of `src/vlc_media_player.rs` into `src/vlc_media_player/software_video.rs`. Re-export and wire through unchanged.
- Introduce `force_hardware: bool` `#[export]` on `VLCMediaPlayer`, default `false`, no consumer yet — the setter just stores the value.
- Add `gpu` cargo feature with `cfg(windows)` default. Empty `src/vlc_media_player/gpu_d3d11/mod.rs` behind `#[cfg(feature = "gpu")]`.
- Add `windows` crate dependency (behind the feature), pinned to a current release.

**Verify**: `cargo build --release` and `cargo build --release --features=gpu` both succeed. `demo/` project plays `test.mp4` byte-identically to the pre-refactor build (visual diff check — same frame at t=1.0s).

### Phase 1 — Adapter matching

No libvlc dependency. Proves the LUID-match chain works against a real Godot D3D12 device.

- Implement `AdapterEnumerator` trait + real DXGI-backed impl in `gpu_d3d11/adapter.rs`.
- Expose a `_debug_get_adapter_luids() -> Dictionary` method on `VLCMediaPlayer` behind `#[cfg(feature = "gpu")]` that returns `{godot_luid: i64, d3d11_luid: i64}` for test probes.
- Hard-fail with a specific error string if Godot is not on D3D12. Integration test asserts the string rather than crashing.

**Verify:**
- *Unit* `src/vlc_media_player/gpu_d3d11/adapter.rs` tests: `find_adapter_by_luid` against a fake enumerator — match-first / match-last / no-match / duplicate-luid.
- *Integration* `demo/tests/phase1_adapter.gd`: launch flag `--rendering-driver d3d12`, assert `godot_luid == d3d11_luid` and both nonzero.
- *Integration* `demo/tests/phase1_wrong_driver.gd`: launch with `--rendering-driver vulkan`, assert the probe returns the expected error rather than crashing.

### Phase 2 — D3D12 import + fence timing spike

The de-risk phase. Three questions gate Phases 4+5:

1. Does `RenderingDevice::texture_create_from_extension` accept an externally-opened `ID3D12Resource*` and produce correct D3D12 state-tracker entries?
2. Does `Texture2Drd::set_texture_rd_rid` accept an RID made by us?
3. Does a `Wait` enqueued from `frame_pre_draw` order *before* the sample of that texture in the same frame?

Also settles ownership: does `texture_create_from_extension` `AddRef` the `ID3D12Resource*`? Determines release ordering in Phase 5.

Spike build (no VLC yet):

- CPU-generate a checkerboard; upload to a D3D11 shared texture (`SHARED_NTHANDLE | SHARED_KEYEDMUTEX`) via `UpdateSubresource`.
- `ID3D11Fence` with `SHARED_NTHANDLE` on the D3D11 side.
- Open the texture handle on Godot's `ID3D12Device`, wrap via `texture_create_from_extension`, assign RID to a `Texture2Drd`, display in a `TextureRect`.
- Open the fence handle on the D3D12 device.
- From a worker thread: sleep 50 ms, rewrite the checkerboard to a different pattern via D3D11, signal fence value `N`.
- Connect to `frame_pre_draw`, enqueue `ID3D12CommandQueue::Wait(fence, N)` on Godot's direct queue.

**Verify:**
- *Integration* `demo/tests/phase2_import.gd`: runs the spike, screenshots the viewport, pixel-perfect compare vs. checkerboard-after fixture. Zero debug-layer warnings.
- *Integration* `demo/tests/phase2_fence_timing.gd`: signals after `frame_pre_draw` returns but before the frame's sample executes. If `Wait` is effective we see the post-signal pattern; otherwise the test fails and triggers the fallback inventory below.
- *Manual* PIX capture: confirm barriers around the imported resource look sane; no `D3D12_MESSAGE_ID_*` warnings on it.

**If the fence-timing test fails**, inventory the options before moving on:
(a) own a compute pre-pass that issues the wait;
(b) GPU `copy_texture` on swap (loses zero-copy within Godot but keeps zero CPU copy);
(c) keyed-mutex acquire/release around the sample — likely requires (a) anyway since Godot's render graph doesn't expose per-external-texture hooks.

### Phase 3 — VLC D3D11 output callbacks

Wire libvlc's D3D11 output into our spike. No Godot import yet; validate via CPU readback.

- Gate the GPU callbacks on `force_hardware == true` in `VLCMediaPlayer::register_player_callbacks`. On failure anywhere in setup, log via `godot_error!` and take the software path.
- Implement `setup_cb` / `update_output_cb` / `swap_cb` / `cleanup_cb` per the architecture section.
- Reuse the D3D11 device created in Phase 1's `adapter.rs`.

**Verify:**
- *Integration* `demo/tests/phase3_cpu_readback.gd`: plays `demo/test.mp4`, after 1 s triggers a CPU-readback helper (copy VLC-owned D3D11 texture to staging → map), asserts average pixel colour in expected range. Proves VLC actually rendered into our texture.
- *Integration* `demo/tests/phase3_resize_event.gd`: forces a size change (`libvlc_video_set_scale` or switching media), asserts exactly one new `{handle, w, h}` arrives on the SPSC queue with the new dimensions.
- *Integration* `demo/tests/phase3_fallback.gd`: sets `force_hardware = true` then artificially sabotages setup (e.g. by launching under Vulkan), asserts the player plays anyway via the software path and the error was logged.

### Phase 4 — End-to-end zero-copy

Connect Phase 3's VLC output to Phase 2's D3D12 import.

- Implement `rd_import.rs` (importer thread, deferred-delete queue).
- Swap the `ImageTexture` inside `VLCMediaPlayer` for a `Texture2Drd` when the GPU backend is active. `get_texture()` upcasts to `Texture2D` so the return type stays stable.
- Keep the child `TextureRect`; point it at the `Texture2Drd`.

**Verify:**
- *Integration* `demo/tests/phase4_end_to_end.gd`: plays `demo/test.mp4` with `force_hardware = true`, renders 30 frames headlessly, screenshots every 5th, asserts non-black content + frame-to-frame variance. Zero debug-layer warnings.
- *Integration* `demo/tests/phase4_no_regression.gd`: same clip with `force_hardware = false`, asserts byte-identical pixels vs. Phase 0 baseline. Proves the GPU branch doesn't leak into the CPU path.

### Phase 5 — Resize, source swap, teardown

Robustness.

- Deferred-delete queue indexed by `Engine::get_frames_drawn()`; free old RIDs after `FRAMES_IN_FLIGHT` (3) frames.
- Release ordering per Phase 2's ownership finding.
- `Drop for VlcMediaPlayer` extended: stop playback, wait for VLC threads, drain importer thread, release D3D11 device last. Editor hot-reload must not leak (tested via D3D12 live-object-count helper).

**Verify:**
- *Integration* `demo/tests/phase5_resize_stress.gd`: loop 100× random resize in `[128, 1920] × [128, 1080]`, assert bounded live-object count + no warnings.
- *Integration* `demo/tests/phase5_source_swap.gd`: 20× `set_media(A)` → play → 10 frames → `set_media(B)` → play → 10 frames. Bounded count, no warnings.
- *Integration* `demo/tests/phase5_teardown.gd`: 20× instantiate VLCMediaPlayer with `force_hardware = true`, play 200 ms, `queue_free`, await a frame. Assert live-object count flat from iteration 5 onward.

### Phase 6 — Docs + ship

- README: document `--rendering-driver d3d12` requirement for the GPU path, `force_hardware` property, CPU fallback behaviour, Windows-only for v1.
- Cargo feature doc in `Cargo.toml` / `lib.rs`.
- Demo scene: add a checkbox toggling `force_hardware` at runtime.

**Verify:**
- *Integration* `demo/tests/phase6_demo_smoke.gd`: opens the demo scene headlessly, plays 5 s under both `force_hardware = true` (on D3D12) and `false`, asserts no warnings and non-black output.
- *Manual* PIX capture on the demo: confirm zero-copy — VLC's `ID3D12Resource` pointer equals the one bound to the sampled SRV.

## Commit discipline

- Each phase lands as one or more commits. A phase isn't done until its Verify block passes.
- Prefer a small series of self-contained commits within a phase; each commit builds and its tests pass.
- Commit prefix: `phase N:` (e.g. `phase 1: adapter LUID match`). Final commit of a phase tagged `gpu-phase-N-done`.
- No commits that mix scope across phases or touch the software-video path except in Phase 0.

## Environment requirements

- **`GODOT_BIN`** env var pointing at a Godot 4.3+ `_console` executable (windowed variant swallows stdout and breaks headless tests).
- **VLC 4.0 nightly** already vendored under `thirdparty/vlc/` — unchanged.
- **MSVC x64 toolchain**, Rust stable — standard godot-rust prereqs.

## Open questions / risks

- Phase 2 gates the design. All three questions (RD acceptance of extension RIDs, D3D12 state tracking on imported resources, `frame_pre_draw` wait ordering) are answered by one spike; if any fails, the Phase 2 fallback inventory kicks in and downstream phases rescope.
- `get_driver_resource(COMMAND_QUEUE, ...)` needs to return a usable `ID3D12CommandQueue*` for our fence wait. If it only returns the queue indirectly (or not at all in 4.3), fallback (a) from Phase 2 becomes mandatory. Will be resolved during Phase 2.
- `texture_create_from_extension` ownership semantics (does it AddRef?) — answered empirically in Phase 2 via 30-frame hold-and-release.
- Extension unload flakiness (D3D11 device released mid-VLC) — caught by Phase 5 teardown test.

## Deliverables for v1

- `cargo build --release` (default features) produces today's binary byte-for-byte.
- `cargo build --release --features=gpu` on Windows produces a binary that plays video on the CPU path when `force_hardware = false` (default) and on the D3D12 zero-copy path when `force_hardware = true` and the player is launched with `--rendering-driver d3d12`.
- Verified zero-copy from VLC's D3D11 output to Godot's D3D12 sampling (Phase 6 PIX capture).
- README updates covering launch flag, `force_hardware`, and fallback behaviour.
