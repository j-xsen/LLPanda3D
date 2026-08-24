# GraphicsEngine

**Source:** `panda/src/display/graphicsEngine.h` (+ `.I`, `.cxx`)
**Inherits:** ReferenceCount **Inherited by:** (none)

The top-level object that drives rendering. An application normally owns
exactly one `GraphicsEngine` (though multiple are legal when separate
synchronicity groups are needed); it creates every `GraphicsOutput` (window or buffer)
via `make_output()`, and a single call to `render_frame()` culls, draws, and
(depending on settings) flips every registered window for that frame. It
also owns the optional cull/draw threading pipeline. `PandaFramework` wraps
exactly one of these — see [../framework/PandaFramework.md](../framework/PandaFramework.md)'s
`get_graphics_engine()`, which lazily creates the engine and schedules an
`"igloop"` task to call `render_frame()` once per frame (see
[../event/AsyncTaskManager.md](../event/AsyncTaskManager.md)).

## Behavior notes

- **`make_output()` tries several strategies in order**, controlled by the
  `flags` bitmask (see [GraphicsPipe.md](GraphicsPipe.md) for the `BF_*`
  flags) and the `prefer-parasite-buffer`/`force-parasite-buffer`/
  `prefer-texture-buffer` config variables: (1) if
  `BF_require_callback_window` is set, always makes a
  [CallbackGraphicsWindow.md](CallbackGraphicsWindow.md); (2) if a
  `ParasiteBuffer` is usable (host given, not requiring a real window/callback
  window, host's framebuffer properties subsume what's requested) and
  `prefer-parasite-buffer` is set and it fits within the host, makes a
  [ParasiteBuffer.md](ParasiteBuffer.md) immediately; (3) if
  `force-parasite-buffer` is set, likewise, even if it doesn't fit width/height
  (parasite still used); (4) otherwise asks `pipe->make_output()` up to 10
  times with an incrementing `retry` counter — this lets the pipe fall back
  through progressively less-capable OpenGL/DirectX capability sets when an
  earlier attempt's window fails to actually open; (5) if all 10 pipe
  attempts fail and a parasite would have been usable, falls back to a
  `ParasiteBuffer` as a last resort. Returns `nullptr` only if every strategy
  fails.
- **A `nullptr` or not-yet-valid `gsg`/`host` is silently downgraded.** If
  the passed-in `gsg` or `host->get_gsg()` isn't valid or `needs_reset()`,
  `make_output()` calls `open_windows()` once to try to bring it up; if it's
  still not valid afterward, the `gsg`/`host` argument is discarded (set to
  `nullptr`) and a fresh one gets created instead — silent, not an error.
- **`make_buffer()` has two overloads with different sharing behavior**
  (documented in `.I`, easy to miss from the header alone): the
  `GraphicsOutput*` (host window) overload adapts to the host's existing
  framebuffer properties and is the preferred way to create an offscreen
  buffer when a window already exists; it maximizes resource sharing.
  The `GraphicsStateGuardian*` overload builds its own `FrameBufferProperties`
  from scratch (forcing off back buffers, stereo, accum, multisamples,
  forced-hardware/software) and is intended for use only when there is no
  existing window to piggyback on — it does a "poorer job of sharing the
  GSG."
- **`render_frame()`'s per-frame sequence** (all under `_public_lock`, so
  `get_render_lock()` held externally blocks a frame from proceeding):
  flush the disk-cache index if due → `open_windows()` (realize any
  pending buffers/windows first) → optionally `do_flip_frame()` if
  `sync-flip` is set and the previous frame hasn't flipped yet → drop any
  windows whose delete flag got set → pre-compute each active
  `DisplayRegion`'s camera scene bounding volume **in the app thread**
  (before the pipeline cycles), specifically so the bounds are more likely
  to still be valid/cached for the cull thread rather than recomputed there
  → release RAM images of textures uploaded last frame → run the `_app`
  `WindowRenderer`'s own cull/draw/window work synchronously → wait for all
  render threads to reach `TS_wait` → cycle the `Pipeline` and tick the
  global `ClockObject` → signal all render threads to begin
  `TS_do_frame` → set `_flip_state` to `FS_flip` if `auto-flip` is set,
  else `FS_draw` (meaning a later `flip_frame()`/`sync_flip` call is
  needed) → optionally yield the timeslice (`yield-timeslice` config, or a
  `Thread::consider_yield()` in non-true-threading builds).
- **Threading model is per-window-list, not per-window-instance.**
  `WindowRenderer` (private nested class) is the unit of scheduling: there is
  one `_app` `WindowRenderer` plus one `RenderThread` (itself a
  `WindowRenderer` subclass that's also a `Thread`) per distinct
  cull/draw/window thread name parsed out of the `GraphicsThreadingModel`
  (see [GraphicsThreadingModel.md](GraphicsThreadingModel.md)). A window is
  added to up to three `WindowRenderer`s' lists — one for its "window" task
  (OS-level resize/placement calls), one for "cull," one for "draw" — chosen
  from `_threading_model.get_cull_name()`/`get_draw_name()` and each stage's
  preferred thread from `GraphicsPipe::get_preferred_window_thread()` (X11
  requires the window task on the app thread; some platforms require it on
  draw because a GL context, once bound in a thread, can't move). In the
  default single-threaded model all three lists are the same (`_app`).
- **The `"-"`-prefixed threading-model variant skips binning.** If
  `_threading_model.get_cull_sorting()` is false (model string starts with
  `-`), a window's rendering goes on the `_cdraw` ("cull-and-draw-together")
  list instead of separate `_cull`/`_draw` lists, and
  `cull_and_draw_together()` culls and immediately issues draw commands per
  `Geom` with no bin-sorting pass — cheaper but loses draw-order/state
  sorting.
- **`open_windows()` runs the window-open handshake twice** (`for (int i =
  0; i < 2; ++i)`) specifically "to allow both cull and draw to process the
  window" — opening a window can require both stages to see it once before
  it's fully realized.
- **GSG cleanup happens opportunistically, not eagerly**, in
  `WindowRenderer::do_frame()`/`do_close()`: after each frame, any GSG in a
  renderer's `_gsgs` set whose `get_ref_count() == 1` (meaning only the
  `WindowRenderer` itself still holds it — no window uses it anymore) is
  closed via `GraphicsPipe::close_gsg()`. This only happens per-frame, so a
  GSG with no more windows can outlive its last window by up to one frame.
- **`extract_texture_data()`, `dispatch_compute()`, and
  `do_get_screenshot()` all use the same "borrow the draw thread"
  pattern**: in a single-threaded setup they call straight through directly; in
  a multithreaded setup they grab the relevant `RenderThread`'s condition
  variable, wait for it to be `TS_wait` (idle between frames), temporarily
  reassign its pipeline stage to the calling thread's stage, signal a
  one-off `TS_do_extract`/`TS_do_compute`/`TS_do_screenshot` state, and
  block until it's done. This means these calls can stall the calling
  thread until the draw thread finishes its current frame.
- **`is_scene_root()` backs `PandaNode::is_scene_root()`** via a static
  function pointer (`PandaNode::set_scene_root_func`) installed the first
  time `get_global_ptr()` creates the global engine — so a node only reports
  itself as a scene root once at least one `GraphicsEngine` has been
  constructed and has an active window/camera pointing at it.
- **`do_cull()`'s view-frustum culling is skippable per the `view-frustum-cull`
  config var** — when disabled, `CullTraverser::set_view_frustum(nullptr)`
  is left in place and everything in the scene graph gets traversed
  regardless of camera frustum, which is useful for debugging but a real
  performance cost.
- **`setup_scene()` detects and refuses singular (non-invertible) net
  transforms** on either the scene root or the camera (e.g. a zero scale
  somewhere in the ancestor chain) — it logs a warning at most once per
  frame (`_singular_warning_last_frame`/`_singular_warning_this_frame`
  latch, to avoid log spam every frame the condition persists) and returns
  `nullptr`, which causes that `DisplayRegion` to be skipped for the frame
  entirely.
- **An inverted window flips more than pixels.** When
  `GraphicsOutput::get_inverted()` is true, `setup_scene()` does more than
  set `SceneSetup::set_inverted(true)`: it also composes the camera's
  `initial_state` with a global cached "invert polygon winding" `RenderState`
  (`get_invert_polygon_state()`, a `CullFaceAttrib::make_reverse()`,
  memoized forever once first requested) — necessary because flipping the
  image also flips the apparent front/back winding of every polygon.
- **Stereo `DisplayRegion`s are never drawn directly.** Both
  `cull_and_draw_together()` and `do_draw()` explicitly skip a
  `DisplayRegion` for which `dr->is_stereo()` is true — a
  [StereoDisplayRegion.md](StereoDisplayRegion.md) is a placeholder that
  still participates in clearing but whose actual left/right eye rendering
  happens through separate constituent `DisplayRegion`s.
- **A `DisplayRegion`'s cull/draw callback fully replaces the normal cull or
  draw step**, not only decorates it — if `DisplayRegion::get_cull_callback()`
  / `get_draw_callback()` returns non-null, `GraphicsEngine` invokes the
  `CallbackObject` instead of running `CullTraverser`/`CullResult::draw()`.
  Before invoking a draw callback it explicitly resets the GSG's state to
  depth-testing disabled and identity transform (`gsg->clear_before_callback()`)
  "since some libraries (eg. Kivy) expect that," and afterward calls
  `gsg->clear_state_and_transform()` because it doesn't trust what state the
  callback left behind.
- **Capability auto-elevation is one-directional and sticky.**
  `auto_adjust_capabilities()` (called once per newly-added GSG from
  `do_add_gsg()`) implements a documented one-way ratchet for global
  capability flags like `Texture::get_textures_power_2()`: global
  capabilities start conservative and only get *elevated* to a more
  aggressive setting once a GSG proves it supports the aggressive mode —
  never lowered again afterward, because textures created under a
  once-elevated setting may already exist and can't be retroactively
  un-elevated.
- **`do_add_window()` stamps an internal monotonic sort tiebreaker**
  (`_internal_sort_index`, a plain incrementing counter) on every window so
  that windows with equal explicit `sort` values still end up ordered by
  creation order in `get_window(n)`, rather than in unspecified order.
- **`remove_window()` also releases shared GSG resources when appropriate.**
  After removing a window, if no other still-active window shares that
  window's GSG's `PreparedGraphicsObjects`, it calls `pgo->release_all()` —
  explicitly to avoid leaking graphics-memory allocations even if some
  stray pointer elsewhere is keeping the GSG object itself alive.
- **`RenderThread::thread_main()`'s inner `switch`** is a straightforward
  state machine driven by `_thread_state` (`ThreadState` enum) and a pair of
  condition variables (`_cv_start`/`_cv_done`) — the app thread sets
  `_thread_state` and signals `_cv_start`, the render thread does the work,
  sets `_thread_state = TS_wait`, and signals `_cv_done`; `TS_terminate`
  closes pending windows/GSGs and exits the loop permanently (`TS_done`).

## API

### Construction

| Signature | Notes |
|---|---|
| `explicit GraphicsEngine(Pipeline *pipeline = nullptr)` | `nullptr` uses `Pipeline::get_render_pipeline()` (the global pipeline). Reads the `threading-model` config var for the initial threading model. |
| `BLOCKING ~GraphicsEngine()` | Calls `remove_all_windows()`. |
| `static GraphicsEngine *get_global_ptr()` | Lazily creates a process-global instance and registers it as the `PandaNode` scene-root oracle. |

### Threading / global settings

| Signature | Notes |
|---|---|
| `void set_threading_model(const GraphicsThreadingModel &)` / `GraphicsThreadingModel get_threading_model() const` | Affects only subsequently-created outputs. Warns and ignores the request if threading isn't supported/compiled in (unless `allow-nonpipeline-threads`). |
| `INLINE const ReMutex &get_render_lock() const` | Held for the full duration of `render_frame()`; acquiring it externally guarantees no frame is mid-render. |
| `INLINE void set_auto_flip(bool)` / `INLINE bool get_auto_flip() const` | See behavior notes on `_flip_state`. Defaults from the `auto-flip` config var. |
| `INLINE void set_portal_cull(bool)` / `INLINE bool get_portal_cull() const` | Toggles portal culling. |
| `INLINE void set_default_loader(Loader *)` / `INLINE Loader *get_default_loader() const` | Assigned to every GSG this engine subsequently creates. |

### Creating outputs

| Signature | Notes |
|---|---|
| `GraphicsOutput *make_output(GraphicsPipe *pipe, const string &name, int sort, const FrameBufferProperties &fb_prop, const WindowProperties &win_prop, int flags, GraphicsStateGuardian *gsg = nullptr, GraphicsOutput *host = nullptr)` | The general entry point. See behavior notes for its fallback strategy. |
| `INLINE GraphicsOutput *make_buffer(GraphicsOutput *host, const string &name, int sort, int x_size, int y_size)` | Preferred convenience form when a window/buffer already exists. |
| `INLINE GraphicsOutput *make_buffer(GraphicsStateGuardian *gsg, const string &name, int sort, int x_size, int y_size)` | Convenience form for when nothing exists yet; shares less. |
| `INLINE GraphicsOutput *make_parasite(GraphicsOutput *host, const string &name, int sort, int x_size, int y_size)` | Forces `GraphicsPipe::BF_require_parasite`. |
| `bool add_window(GraphicsOutput *window, int sort)` | Registers an already-constructed `GraphicsOutput` (advanced/custom-window use only — `make_output()` normally does this). |

### Window list

| Signature | Notes |
|---|---|
| `bool remove_window(GraphicsOutput *window)` | Closes and deregisters; releases shared GSG resources if orphaned. Doesn't stop any render threads. |
| `BLOCKING void remove_all_windows()` | Closes everything, terminates all render threads, flushes the bam cache, stops async task/vertex-paging threads. Called by the destructor. |
| `void reset_all_windows(bool swapchain)` | DirectX 8 only: forces framebuffer recreation on every window. |
| `bool is_empty() const` / `int get_num_windows() const` / `GraphicsOutput *get_window(int n) const` | `get_window()` returns windows in sorted (by `sort`, then creation order) order. |

### Frame control

| Signature | Notes |
|---|---|
| `BLOCKING void render_frame()` | Cull + draw (+ flip if `auto_flip`) every registered window for one frame. See behavior notes for the full sequence. |
| `BLOCKING void open_windows()` | Forces pending window opens/closes without rendering a frame. |
| `BLOCKING void sync_frame()` | Waits for in-flight draw threads to finish submitting (not necessarily GPU-finished). |
| `BLOCKING void ready_flip()` | OpenGL-only: forces a GPU pixel readback to ensure drawing has actually completed before flip, not just been submitted. No-op on other APIs. |
| `BLOCKING void flip_frame()` | Waits for drawing to finish, then flips every window. |

### Texture / compute utilities

| Signature | Notes |
|---|---|
| `bool extract_texture_data(Texture *tex, GraphicsStateGuardian *gsg)` | Pulls the GPU-side image back into `tex`'s RAM image. Loads the texture to the GSG first if needed. May block on the draw thread. |
| `void dispatch_compute(const LVecBase3i &work_groups, const ShaderAttrib *sattr, GraphicsStateGuardian *gsg)` | One-off compute-shader dispatch outside the scene graph (no `ComputeNode` needed). |
| `void texture_uploaded(Texture *tex)` | Called by the GSG after a successful upload; queues the RAM image for release at end of frame (deferred so multiple GSGs sharing the texture within a frame all get a chance first). |
| `PT(Texture) do_get_screenshot(DisplayRegion *region, GraphicsStateGuardian *gsg)` | Internal; called by `DisplayRegion::do_get_screenshot()`. |

## See also

- [../framework/PandaFramework.md](../framework/PandaFramework.md) — owns and drives one `GraphicsEngine`.
- [GraphicsPipe.md](GraphicsPipe.md), [GraphicsOutput.md](GraphicsOutput.md), [GraphicsWindow.md](GraphicsWindow.md) — what `make_output()` creates and on what.
- [GraphicsStateGuardian.md](GraphicsStateGuardian.md) — the renderer object each output pairs with; capability auto-elevation is keyed off it.
- [DisplayRegion.md](DisplayRegion.md), [StereoDisplayRegion.md](StereoDisplayRegion.md) — the per-camera render targets `render_frame()` culls/draws.
- [GraphicsThreadingModel.md](GraphicsThreadingModel.md) — the string format parsed by `set_threading_model()`.
- [README.md](README.md) — module-wide config variables and the Engine→Pipe→Output→Region shared concept.

## Uncertain / worth double-checking

- The exact semantics of `ready_flip()` (documented as "will only work in
  OpenGL right now... reads a single pixel to force the card to finish") is
  taken directly from the header comment; I did not trace into the GL GSG
  backend (out of scope for this module) to verify the pixel-read mechanism.
