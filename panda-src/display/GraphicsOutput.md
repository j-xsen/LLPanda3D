# GraphicsOutput

**Source:** `panda/src/display/graphicsOutput.h` (+ `.I`, `.cxx`)
**Inherits:** GraphicsOutputBase (external), [DrawableRegion](DrawableRegion.md) **Inherited by:** [GraphicsBuffer](GraphicsBuffer.md), [ParasiteBuffer](ParasiteBuffer.md), [GraphicsWindow](GraphicsWindow.md)

Base class for "the result of a frame of rendering" — either a real
`GraphicsWindow` on the desktop or an offscreen `GraphicsBuffer`/
`ParasiteBuffer`. Owns the associated `GraphicsStateGuardian` (one GSG per
output; a GSG may serve multiple outputs), the framebuffer properties, the
list of `DisplayRegion`s carved out of it, optional render-to-texture
targets, and the active/sort bookkeeping `GraphicsEngine` uses to decide
render order. Constructors are `protected` — real instances only ever come
from `GraphicsEngine::make_output()` (see [GraphicsEngine.md](GraphicsEngine.md)).
`GraphicsOutput` inherits `TypedWritableReferenceCount` (via
`GraphicsOutputBase`) rather than plain `TypedReferenceCount` specifically
so instances can be passed as event parameters, even though outputs are
never actually written to bam files.

## Behavior notes

- **Every `GraphicsOutput` gets one hidden "overlay" `DisplayRegion` at
  construction time** (`_overlay_display_region`, created via
  `make_mono_display_region(0,1,0,1)` and immediately set inactive with
  scissor disabled). It covers the whole output but is never rendered
  through directly — it exists purely as the target for whole-window
  operations: `clear()`, `save_screenshot()`/`get_screenshot()`, and the
  `RenderBuffer` type lookups. `get_overlay_display_region()` exposes it
  read-only; `set_overlay_display_region()` lets you replace it but does
  **not** remove the old one for you — you must separately call
  `remove_display_region()` on it.
- **Default clear behavior on construction**: every new `GraphicsOutput`
  starts with color, depth, and stencil clear-active all `true` and the
  clear color set from the `background-color` config variable (see
  [README.md](README.md#config-variables-config_displayh)) — a fresh output
  clears to the configured gray by default unless you turn clears off.
- **Stereo mode is decided once, at construction, from config**, via the
  `default_stereo_flags` constructor parameter (which `GraphicsWindow`
  passes `true` and `ParasiteBuffer` passes `false`, so a parasite buffer
  never independently red-blue/side-by-side stereo-izes itself — it just
  mirrors its host). `_red_blue_stereo` and `_side_by_side_stereo` are only
  turned on if the framebuffer itself doesn't already claim true hardware
  stereo (`!fb_prop.is_stereo()`). `is_stereo()` is true if *any* of the
  three mechanisms (red-blue, side-by-side, or real hardware stereo) is
  active.
- **`make_display_region()` picks mono vs. stereo automatically** based on
  `is_stereo()` and the `default-stereo-camera` config variable — call
  `make_mono_display_region()` or `make_stereo_display_region()` explicitly
  to bypass the auto-detection. In side-by-side stereo mode, even
  "mono" regions actually become a `StereoDisplayRegion` with both eyes set
  to `Lens::SC_mono`, because side-by-side mode requires drawing every
  region twice (once per half of the window) regardless of whether it's
  conceptually mono content.
- **`make_stereo_display_region()`'s two eyes get separately dimensioned
  sub-rectangles when side-by-side stereo is active** — each eye's requested
  `dimensions` are rescaled into the window's configured
  `sbs_left_dimensions`/`sbs_right_dimensions` sub-region rather than both
  eyes sharing the same rectangle. `swap_eyes` (config var or
  `set_swap_eyes()`) swaps which physical half gets which `DisplayRegion`
  after they're created. When *not* side-by-side, both eyes get identical
  dimensions but the right eye additionally inherits the left/overlay's
  clear-depth/clear-stencil-active flags (on the assumption the two eyes
  share one depth buffer, so only one of them needs to actually clear it —
  but the right eye is made to clear too, redundantly, in this branch;
  reread if depth-buffer-sharing semantics matter for your use case).
- **Render-to-texture is a per-output list, not a single slot** —
  `add_render_texture()` can be called multiple times to bind several
  textures (e.g. one for color, one for depth) to the same output
  simultaneously, tagged with a `RenderTexturePlane` and a `RenderTextureMode`.
  `setup_render_texture()` is the old deprecated single-texture interface
  kept for compatibility, implemented in terms of `clear_render_textures()`
  + `add_render_texture()`.
- **`add_render_texture()`'s default-bitplane inference** picks
  `RTP_depth_stencil` for `F_depth_stencil` format textures, `RTP_depth` for
  any `F_depth_component*` format, and `RTP_color` for everything else —
  and it *overwrites* the texture's format/component-type to match whatever
  bitplane it ends up bound to (e.g. handing it an `F_rgba` texture and
  requesting `RTP_depth_stencil` silently turns it into an `F_depth_stencil`
  texture). It also unconditionally clears the texture's compression
  setting and RAM image. If `RTM_bind_or_copy`/`RTM_bind_layered` was
  requested but the GSG or the `support-render-texture` config var doesn't
  allow binding, the mode silently downgrades to `RTM_copy_texture` — except
  `RTM_bind_layered`, which has no copy fallback (layered/cube/3-D-texture
  targets can't be produced by a simple framebuffer copy) and just logs an
  error instead.
- **`RenderTextureMode` distinguishes "direct" vs. "copy" render-to-texture**:
  `RTM_bind_or_copy` (try direct GPU binding, fall back to copy),
  `RTM_copy_texture`/`RTM_copy_ram` (explicit per-frame copy to
  texture-memory or system RAM), `RTM_triggered_copy_texture`/
  `RTM_triggered_copy_ram` (copy only after an explicit `trigger_copy()`
  call — useful for expensive offscreen renders you don't need every
  frame), and `RTM_bind_layered` (direct binding to a specific layer of a
  cube map/3-D texture/texture array, selected via a geometry shader — no
  copy fallback exists).
- **`make_texture_buffer()` is the "just give me a texture" convenience
  entry point** — it builds minimal `FrameBufferProperties` (1 color bit, 1
  alpha bit, 1 depth bit — i.e. "I'll take whatever depth/color/alpha the
  backend can give me") unless you supply your own, requests
  `GraphicsPipe::BF_refuse_window` (so `GraphicsEngine::make_output()` is
  forced to produce an offscreen buffer or `ParasiteBuffer` rather than a
  real window — see [GraphicsPipe.md](GraphicsPipe.md) for the `BF_*` flag
  meanings), and forces the copy to go through system RAM if the new
  buffer's GSG doesn't share `PreparedGraphicsObjects` with this output's
  GSG (i.e. textures created on one GSG generally can't be handed directly
  to a different GSG's context). The actual choice of buffer strategy
  (direct GraphicsBuffer vs. ParasiteBuffer vs. RAM-copy fallback) is made
  by `GraphicsEngine::make_output()`, steered by the `prefer-texture-buffer`
  /`prefer-parasite-buffer`/`force-parasite-buffer`/`prefer-single-buffer`
  config variables (see [README.md](README.md#config-variables-config_displayh)).
- **`make_cube_map()` builds a complete 6-camera rig**, not just a texture —
  it creates one `PerspectiveLens(90,90)` shared by six `Camera`s (one per
  cube face, aimed via `look_at()` using hardcoded axis-aligned
  look/up vectors), parents them all to the caller-supplied `camera_rig`
  NodePath (after stripping any inherited rotation from it via a
  `CompassEffect`, so the rig stays world-axis-aligned regardless of where
  it's parented), and creates six `DisplayRegion`s on the resulting buffer,
  each tagged with `set_target_tex_page(i)` so the render loop knows which
  cube face each region writes to. It disables the buffer's own
  whole-output clear in favor of per-region clearing (via
  `copy_clear_settings()` inherited from `DrawableRegion`), and clamps
  `size` to the GSG's `get_max_cube_map_dimension()` unless rendering to
  RAM only.
- **`change_scenes()` is the cube-map/multi-page-texture page-switch
  hook**, called by `GraphicsEngine` between `DisplayRegion`s when the
  target texture page changes. In bind/layered render-to-texture mode it
  just calls `select_target_tex_page()` to redirect the backend to the new
  page for the *next* frame's draw; in copy-to-texture mode it must
  immediately copy the framebuffer contents rendered so far into the *old*
  page's texture, because that data will be gone once drawing continues
  into the new page.
- **`get_delete_flag()` refuses to report true while any bound
  render-to-texture texture is still externally referenced** — even after
  `GraphicsEngine::remove_window()` marks the buffer for deletion
  (`prepare_for_deletion()` sets `_delete_flag = true` and moves any
  `RTM_bind_or_copy`/`RTM_bind_layered` textures into `_hold_textures` as
  weak pointers), the buffer is kept alive until every one of those weak
  pointers reports invalid — i.e. until nothing else holds a `PT(Texture)`
  to the rendered result. This is what prevents "close a render-to-texture
  buffer while a still-visible model is using its texture" from silently
  corrupting the texture.
- **`get_fb_size()`/`get_fb_x_size()`/`get_fb_y_size()` differ from
  `get_size()`/`get_x_size()`/`get_y_size()` only when pixel-zoom is in
  effect** (see [DrawableRegion.md](DrawableRegion.md)) — `get_size()` is
  the visible/logical size, `get_fb_size()` is `size * get_pixel_factor()`,
  clamped to a minimum of `1` per axis.
- **Side-by-side sizing (`get_sbs_left_size()`/`get_sbs_right_size()` and
  the x/y variants) derive from `get_size()` scaled by the configured
  `sbs_left_dimensions`/`sbs_right_dimensions` fractional rectangles** —
  when side-by-side stereo isn't enabled these simply equal `get_size()`.
- **`is_active()` has several independent ways to be true beyond the
  obvious active flag**: a one-shot output is active only during its
  designated frame (`_one_shot_frame` matches the current global frame
  count); an output with any clear flag active is always considered active
  (so it still gets cleared every frame even with zero display regions); an
  output with a pending `trigger_copy()` future is active; and otherwise
  it's active only if it has at least one currently-active `DisplayRegion`
  (lazily recomputed via `determine_display_regions()` when the "stale"
  flag is set).
- **`set_sort()` doesn't just assign a member** — it delegates to
  `GraphicsEngine::set_window_sort()` so the engine can re-sort its
  internal ordering; the plain `_sort` member is only ever mutated through
  that path, never directly.
- **The whole `_textures` list, the `_active` flag, and the active-regions
  cache live in a `PipelineCycler<CData>`** (see `GraphicsEngine.md` for the
  general cull/draw pipelining concept) — most of `GraphicsOutput`'s other
  state is *not* cycled, only these frame-sensitive fields, since they can
  legitimately be read from a different pipeline stage than they're written.

## API

### Identity / association

| Signature | Notes |
|---|---|
| `INLINE GraphicsStateGuardian *get_gsg() const` | May be `nullptr` before first frame or after close. |
| `INLINE GraphicsPipe *get_pipe() const` | |
| `INLINE GraphicsEngine *get_engine() const` | |
| `INLINE const std::string &get_name() const` | |
| `virtual GraphicsOutput *get_host()` | Self, except on `ParasiteBuffer` where it returns the underlying host output. |

### Render-to-texture

| Signature | Notes |
|---|---|
| `enum RenderTextureMode { RTM_none, RTM_bind_or_copy, RTM_copy_texture, RTM_copy_ram, RTM_triggered_copy_texture, RTM_triggered_copy_ram, RTM_bind_layered }` | See behavior notes. |
| `INLINE int count_textures() const` / `INLINE bool has_texture() const` | |
| `virtual INLINE Texture *get_texture(int i=0) const` | |
| `INLINE RenderTexturePlane get_texture_plane(int i=0) const` / `INLINE RenderTextureMode get_rtm_mode(int i=0) const` | |
| `void clear_render_textures()` | Fires `"render-texture-targets-changed"` event. |
| `void add_render_texture(Texture *tex, RenderTextureMode mode, RenderTexturePlane bitplane=RTP_COUNT)` | `RTP_COUNT` sentinel means "infer from texture format". Fires the same event. |
| `void setup_render_texture(Texture *tex, bool allow_bind, bool to_ram)` | Deprecated single-texture form. |
| `INLINE AsyncFuture *trigger_copy()` | Triggers `RTM_triggered_copy_*` modes at end of next frame; returns an awaitable future. |

### Size / validity

| Signature | Notes |
|---|---|
| `INLINE const LVecBase2i &get_size() const` / `get_x_size()` / `get_y_size()` | Not thread-safe for a live window; use `get_properties()` for that. |
| `INLINE LVecBase2i get_fb_size() const` / `get_fb_x_size()` / `get_fb_y_size()` | Scaled by `get_pixel_factor()`. |
| `INLINE LVecBase2i get_sbs_left_size() const` / `get_sbs_left_x_size()` / `get_sbs_left_y_size()` | |
| `INLINE LVecBase2i get_sbs_right_size() const` / `get_sbs_right_x_size()` / `get_sbs_right_y_size()` | |
| `INLINE bool has_size() const` / `INLINE bool is_valid() const` / `INLINE bool is_nonzero_size() const` | `is_valid()` requires both `_is_valid` and nonzero size. |
| `void set_size_and_recalc(int x, int y)` | Recomputes all `DisplayRegion` pixel rectangles + the texture-card geometry. |

### Activity / lifecycle flags

| Signature | Notes |
|---|---|
| `void set_active(bool)` / `virtual bool is_active() const` | See behavior notes for the several ways this can be true. |
| `void set_one_shot(bool)` / `bool get_one_shot() const` | Auto-deactivates after the frame it was set in; still requires an explicit `remove_window()` to free it. |
| `INLINE void clear_delete_flag()` / `bool get_delete_flag() const` | See behavior notes — texture-hold logic. |
| `virtual void set_sort(int)` / `INLINE int get_sort() const` | Delegates to `GraphicsEngine::set_window_sort()`. |
| `INLINE void set_child_sort(int)` / `INLINE void clear_child_sort()` / `INLINE int get_child_sort() const` | Sort assigned to buffers this output creates via `make_texture_buffer()`; defaults to `get_sort() - 1`. |

### Stereo

| Signature | Notes |
|---|---|
| `void set_inverted(bool)` / `INLINE bool get_inverted() const` | Upside-down/mirrored render, for DX/buggy-driver compensation. Recomputes all `DisplayRegion` pixel rects when toggled. |
| `INLINE void set_swap_eyes(bool)` / `INLINE bool get_swap_eyes() const` | |
| `INLINE void set_red_blue_stereo(bool, unsigned int left_mask, unsigned int right_mask)` / getters | Anaglyph fallback stereo. |
| `void set_side_by_side_stereo(bool)` / `void set_side_by_side_stereo(bool, const LVecBase4 &left, const LVecBase4 &right)` / getters | |
| `INLINE bool is_stereo() const` | True if red-blue, side-by-side, or true hardware stereo. |
| `INLINE const FrameBufferProperties &get_fb_properties() const` | |

### DisplayRegion management

| Signature | Notes |
|---|---|
| `INLINE DisplayRegion *make_display_region()` / `(l,r,b,t)` / `(const LVecBase4 &)` | Auto mono-vs-stereo. |
| `INLINE DisplayRegion *make_mono_display_region()` / `(l,r,b,t)` / `(const LVecBase4 &)` | Forces mono (or side-by-side-doubled mono). |
| `INLINE StereoDisplayRegion *make_stereo_display_region()` / `(l,r,b,t)` / `(const LVecBase4 &)` | Always stereo. |
| `bool remove_display_region(DisplayRegion *)` | Also removes both eyes if given a stereo region. |
| `void remove_all_display_regions()` | Keeps only the overlay region. |
| `INLINE DisplayRegion *get_overlay_display_region() const` / `void set_overlay_display_region(DisplayRegion *)` | See behavior notes. |
| `int get_num_display_regions() const` / `PT(DisplayRegion) get_display_region(int n) const` | Plus `MAKE_SEQ` iteration (`get_display_regions`). |
| `int get_num_active_display_regions() const` / `PT(DisplayRegion) get_active_display_region(int n) const` | Plus `MAKE_SEQ` iteration (`get_active_display_regions`). |

### Offscreen convenience buffers

| Signature | Notes |
|---|---|
| `GraphicsOutput *make_texture_buffer(const string &name, int x_size, int y_size, Texture *tex=nullptr, bool to_ram=false, FrameBufferProperties *fbp=nullptr)` | See behavior notes. |
| `GraphicsOutput *make_cube_map(const string &name, int size, NodePath &camera_rig, DrawMask camera_mask=..., bool to_ram=false, FrameBufferProperties *fbp=nullptr)` | See behavior notes. |

### Screenshots

| Signature | Notes |
|---|---|
| `INLINE static Filename make_screenshot_filename(const string &prefix="screenshot")` | Uses `screenshot-filename`/`screenshot-extension` config vars. |
| `INLINE Filename save_screenshot_default(const string &prefix="screenshot")` / `INLINE bool save_screenshot(const Filename &, const string &image_comment="")` | Delegate to the overlay `DisplayRegion`. |
| `INLINE bool get_screenshot(PNMImage &)` / `INLINE PT(Texture) get_screenshot()` | Delegate to the overlay `DisplayRegion`. |

### Misc

| Signature | Notes |
|---|---|
| `NodePath get_texture_card()` | Fresh `PandaNode` each call, textured with the first non-depth-stencil render texture; auto-updated to match output size. |
| `virtual bool share_depth_buffer(GraphicsOutput *)` / `virtual void unshare_depth_buffer()` | Base implementations are no-ops returning `false`; meaningful overrides live in per-GSG-backend subclasses outside this module. |
| `virtual bool get_supports_render_texture() const` | Base: `false`. |
| `virtual bool flip_ready() const` | |

### Called by GraphicsEngine only (not for general app use)

| Signature | Notes |
|---|---|
| `virtual void request_open()` / `virtual void request_close()` / `virtual void set_close_now()` | |
| `virtual void reset_window(bool swapchain)` / `virtual void clear_pipe()` | |
| `virtual void clear(Thread *)` | Whole-output clear via the overlay region; draw-thread only. |
| `virtual bool begin_frame(FrameMode, Thread *)` / `virtual void end_frame(FrameMode, Thread *)` | Draw-thread only. Base `begin_frame` always returns `false` (a plain `GraphicsOutput` never actually renders — only its subclasses do). |
| `enum FrameMode { FM_render, FM_parasite, FM_refresh }` | |
| `void change_scenes(DisplayRegionPipelineReader *)` | See behavior notes. |
| `virtual void select_target_tex_page(int page)` | Cube-map/multi-page backend hook; no-op at this base level. |
| `virtual void begin_flip()` / `virtual void ready_flip()` / `virtual void end_flip()` | App/main-thread only. |
| `virtual void process_events()` | Window-thread only. |

## See also

- [DrawableRegion.md](DrawableRegion.md) — clear-flag/value API and pixel-zoom mechanism inherited here.
- [GraphicsBuffer.md](GraphicsBuffer.md), [ParasiteBuffer.md](ParasiteBuffer.md) — the two offscreen subclasses.
- [GraphicsWindow.md](GraphicsWindow.md) — the onscreen subclass.
- [GraphicsEngine.md](GraphicsEngine.md) — creates all `GraphicsOutput`s via `make_output()`, drives `begin_frame`/`end_frame`/flip.
- [DisplayRegion.md](DisplayRegion.md) — the sub-regions created by `make_display_region()` and friends.
- [GraphicsStateGuardian.md](GraphicsStateGuardian.md) — the `_gsg` this output renders through.
- [FrameBufferProperties.md](FrameBufferProperties.md) — `_fb_properties`, and the minimal properties `make_texture_buffer()` requests by default.
- [README.md](README.md) — config variables referenced above (`background-color`, `prefer-*-buffer`, `screenshot-*`, stereo defaults).
