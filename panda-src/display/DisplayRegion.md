# DisplayRegion

**Source:** `panda/src/display/displayRegion.h` (+ `.I`, `.cxx`)
**Inherits:** TypedReferenceCount, [DrawableRegion](DrawableRegion.md) **Inherited by:** [StereoDisplayRegion](StereoDisplayRegion.md)

A rectangular sub-region of a [GraphicsOutput](GraphicsOutput.md) (window or
buffer) that renders a scene through a `Camera`. Usually one `DisplayRegion`
covers the whole window, but you can create more to split a window (split
screen, side-by-side stereo, picture-in-picture), or stack them like panes
of glass (layering a 2-d GUI over a 3-d scene, at a higher `sort`).
Constructor is `protected` — created only via
`GraphicsOutput::make_display_region()`/`make_mono_display_region()`/
`make_stereo_display_region()`, which is why `WindowFramework::open_window()`
(see [../framework/WindowFramework.md](../framework/WindowFramework.md))
calls those rather than constructing one directly.

## Behavior notes

- **Pipelined state via `CData`.** Nearly every setter (`set_dimensions`,
  `set_camera`, `set_active`, `set_sort`, `set_stereo_channel`, etc.) writes
  through a `PipelineCycler<CData>` rather than a plain member — the class
  comment on several setters warns "don't call this in a downstream thread
  unless you don't mind it blowing away other changes... in an upstream
  thread." This only matters when a multi-stage `GraphicsThreadingModel` is
  in use; under the (overwhelmingly common) single-threaded default, it's
  invisible. A second, separate cycler (`_cycler_cull`) holds only the
  per-frame cull result (`CullResult`/`SceneSetup`), deliberately kept apart
  from the heavier, rarely-changing `CData` so cull doesn't have to copy
  everything each frame.
- **`get_num_regions()`/`set_num_regions()` — an obscure multi-viewport
  feature.** Normally 1. If set higher, a geometry shader can select which
  numbered sub-rectangle (`gl_ViewportIndex` in GLSL) a given output stream
  goes to — an advanced fast-multi-view technique, not ordinary usage.
  Nearly the whole per-dimension/per-pixel API (`get_dimensions`,
  `get_pixel_width`, etc.) takes an optional region index `i` for this
  reason, defaulting to `0`.
- **Dimensions are `[0,1]` fractions of the window, y-up; pixel conversion
  respects window inversion.** `do_compute_pixels()` maps `_dimensions`
  (left, right, bottom, top, with `(0,0)` = lower-left) to `_pixels`
  (OpenGL-style, bottom-up) and separately to `_pixels_i` (DirectX-style,
  top-down, via `get_region_pixels_i()`) — and if the owning window has
  `get_inverted()` set (see `window-inverted` in the module README), the
  vertical mapping flips so the two pixel-space conventions still line up
  correctly on an inverted window.
- **A `DisplayRegion`'s camera is looked up by `NodePath`, not raw pointer**
  (`set_camera(const NodePath&)`), so which *instance* of a possibly-shared
  `Camera` node is used is unambiguous even if the same `Camera` object
  appears multiple times in the scene graph. Setting a new camera calls
  `remove_display_region(this)` on the old `Camera` and
  `add_display_region(this)` on the new one — one `Camera` can be shared by
  multiple `DisplayRegion`s (that's exactly how `StereoDisplayRegion`'s left
  and right eyes typically share one camera with different
  `stereo_channel`s).
- **`set_stereo_channel()` resets `tex_view_offset` as a side effect** — to
  `1` for `Lens::SC_right`, `0` otherwise — so a stereo setup's per-eye
  texture-view selection tracks the channel automatically unless you
  override it afterward with `set_tex_view_offset()`.
- **`set_target_tex_page()`/`set_cube_map_index()` (the latter deprecated,
  forwards to the former) select which face/page of a multi-page render
  target (cube map, multiview texture) this region renders into**; a
  normal region uses page `-1`.
- **Cull and draw callbacks *replace*, not augment, the normal traversal.**
  `set_cull_callback()`/`set_draw_callback()` install a `CallbackObject`
  that is called *instead of* the built-in cull/draw step; the callback
  must call `cbdata->upcall()` itself if it wants the normal behavior to
  still happen (see the two callback-data subsections below). The cull
  callback fires before this region's cull traversal starts (on the cull
  thread); the draw callback fires after the viewport and this region's
  clears are already applied but before `begin_scene()` (on the draw
  thread).
- **`get_screenshot()` may hop threads.** It compares the GSG's
  `threading_model.get_draw_stage()` against the calling thread's pipeline
  stage; if they don't match (i.e. you're not already on the draw thread)
  it delegates to `GraphicsEngine::do_get_screenshot()` to run there instead
  of grabbing framebuffer contents from the wrong thread.
- **`supports_pixel_zoom()` requires both color and depth clearing to be
  active** on top of the window itself supporting pixel zoom — a region
  with clearing disabled silently can't be zoomed even if the window can.
- **`make_screenshot_filename()`'s pattern language** (used by
  `save_screenshot_default()`) recognizes `%~p` (prefix), `%~f` (frame
  count), `%~e` (extension from `screenshot-extension`), and otherwise
  passes the code through `strftime()`, replacing spaces/colons/slashes in
  the result with `-` and stripping newlines — so a `%Y-%m-%d %H:%M:%S`
  style code becomes filesystem-safe automatically. Pattern default and
  extension default live in the module README's config table
  (`screenshot-filename`, `screenshot-extension`).
- **`get_cull_traverser()` lazily default-constructs a `CullTraverser`** the
  first time it's needed if `set_cull_traverser()` was never called.
- **`DisplayRegionPipelineReader`** is a separate, lightweight, non-copyable
  helper (not a `DisplayRegion` subclass) that snapshots one pipeline
  stage's `CData` for the duration of its scope — used internally by the
  cull/draw machinery (and by `GraphicsStateGuardian::prepare_display_region()`)
  to read a consistent set of `DisplayRegion` properties without repeatedly
  re-locking the cycler. Ordinary application code reads `DisplayRegion`
  properties directly through the class's own getters instead.

## API

### Dimensions / pixels

| Signature | Notes |
|---|---|
| `INLINE int get_num_regions() const` / `INLINE void set_num_regions(int i)` | Multi-viewport count; see behavior notes. |
| `get_dimensions(...)` overloads, `INLINE LVecBase4 get_dimensions(int i=0) const` | `[0,1]` fractional coordinates. |
| `INLINE PN_stdfloat get_left/right/bottom/top(int i=0) const` | |
| `set_dimensions(...)` overloads, `virtual void set_dimensions(int i, const LVecBase4 &)` | |
| `INLINE int get_pixel_width/get_pixel_height(int i=0) const` / `INLINE LVecBase2i get_pixel_size(int i=0) const` | |
| `get_pixels(...)` / `get_region_pixels(...)` / `get_region_pixels_i(...)` | Bottom-up vs. DirectX-style top-down pixel rects; see behavior notes. |
| `void compute_pixels()` / `compute_pixels_all_stages()` / `compute_pixels(x,y)` / `compute_pixels_all_stages(x,y)` | Normally called automatically when the window resizes or dimensions change. |

### Window / pipe / camera

| Signature | Notes |
|---|---|
| `INLINE GraphicsOutput *get_window() const` | |
| `GraphicsPipe *get_pipe() const` | Forwards to the window's pipe. |
| `virtual bool is_stereo() const` | `false` here; `true` on `StereoDisplayRegion`. |
| `virtual void set_camera(const NodePath &camera)` / `INLINE NodePath get_camera(Thread* = current) const` | |
| `void set_lens_index(int index)` / `INLINE int get_lens_index() const` | Selects among a multi-lens camera. |

### Active / sort / stereo / misc flags

| Signature | Notes |
|---|---|
| `virtual void set_active(bool)` / `INLINE bool is_active() const` | Inactive regions render nothing. |
| `virtual void set_sort(int)` / `INLINE int get_sort() const` | Regions render lowest-sort-first; `operator<` compares sort. |
| `virtual void set_stereo_channel(Lens::StereoChannel)` / `INLINE Lens::StereoChannel get_stereo_channel() const` | `SC_mono`/`SC_left`/`SC_right` on a plain `DisplayRegion`; `SC_stereo` only valid on a `StereoDisplayRegion`. |
| `virtual void set_tex_view_offset(int)` / `INLINE int get_tex_view_offset() const` | |
| `virtual void set_incomplete_render(bool)` / `INLINE bool get_incomplete_render() const` | Combines with the GSG's own flag — see `allow-incomplete-render` in the module README. |
| `virtual void set_texture_reload_priority(int)` / `INLINE int get_texture_reload_priority() const` | Higher = this region's async texture reloads are serviced first. |
| `virtual void set_cull_traverser(CullTraverser*)` / `CullTraverser *get_cull_traverser()` | |
| `INLINE void set_cube_map_index(int)` (deprecated) / `virtual void set_target_tex_page(int)` / `INLINE int get_target_tex_page() const` | |
| `INLINE void set_scissor_enabled(bool)` / `INLINE bool get_scissor_enabled() const` | Default `true` except on the internal overlay region. |

### Callbacks

| Signature | Notes |
|---|---|
| `INLINE void set_cull_callback(CallbackObject*)` / `clear_cull_callback()` / `get_cull_callback() const` | See behavior notes. |
| `INLINE void set_draw_callback(CallbackObject*)` / `clear_draw_callback()` / `get_draw_callback() const` | See behavior notes. |

### Screenshots

| Signature | Notes |
|---|---|
| `static Filename make_screenshot_filename(const string &prefix="screenshot")` | |
| `Filename save_screenshot_default(const string &prefix="screenshot")` | |
| `bool save_screenshot(const Filename&, const string &image_comment="")` | |
| `bool get_screenshot(PNMImage&)` / `PT(Texture) get_screenshot()` | |

### Cull-result introspection

| Signature | Notes |
|---|---|
| `void clear_cull_result()` | |
| `virtual PT(PandaNode) make_cull_result_graph()` | Builds a debug scene graph (one node per bin, `GeomNode`s under it) reflecting the last cull pass — for tools/debugging, not real rendering. |

## Subsection: DisplayRegionCullCallbackData

**Source:** `panda/src/display/displayRegionCullCallbackData.h/.I/.cxx`
**Inherits:** CallbackData

Passed to a `DisplayRegion`'s cull callback (`set_cull_callback`).
`get_cull_handler()` returns the `CullHandler` that accepts objects for
drawing; `get_scene_setup()` returns the current camera/scene info. Calling
`upcall()` performs the normal cull traversal (`DisplayRegion::do_cull()`)
that the callback preempted.

## Subsection: DisplayRegionDrawCallbackData

**Source:** `panda/src/display/displayRegionDrawCallbackData.h/.I/.cxx`
**Inherits:** CallbackData

Passed to a `DisplayRegion`'s draw callback (`set_draw_callback`).
`get_cull_result()` returns the `CullResult` (list of objects to draw) built
by cull; `get_scene_setup()` mirrors the cull callback data. Calling
`upcall()` performs the normal draw behavior: for a stereo placeholder
region it does nothing (the real drawing happens on the left/right eye
regions); otherwise it calls `gsg->set_scene()`, clears state/transform, and
runs `begin_scene()` / `_cull_result->draw()` / `end_scene()`.

## Usage

```cpp
// Split a window's DisplayRegion into a picture-in-picture overlay.
DisplayRegion *main_dr = window->get_display_region_3d();
main_dr->set_sort(0);

PT(GraphicsOutput) out = window->get_graphics_output();
DisplayRegion *pip_dr = out->make_display_region(0.65f, 0.98f, 0.65f, 0.98f);
pip_dr->set_camera(minimap_camera);
pip_dr->set_sort(10);  // drawn after (on top of) the main region
```

## See also

- [GraphicsOutput.md](GraphicsOutput.md) — owns and creates `DisplayRegion`s.
- [DrawableRegion.md](DrawableRegion.md) — shared clear-flags base.
- [StereoDisplayRegion.md](StereoDisplayRegion.md) — left/right-eye pair wrapper.
- [GraphicsEngine.md](GraphicsEngine.md) — drives the cull/draw traversal each `DisplayRegion` participates in.
- [../framework/WindowFramework.md](../framework/WindowFramework.md) — `get_display_region_2d()`/`get_display_region_3d()` wrap this class.
