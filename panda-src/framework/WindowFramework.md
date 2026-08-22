# WindowFramework

**Source:** `panda/src/framework/windowFramework.h` (+ `.I`, `.cxx`)
**Inherits:** TypedWritableReferenceCount **Inherited by:** (none in this module; application-specific subclasses possible via `PandaFramework::make_window_framework()`)

Encapsulates everything associated with one open window, or one display
region within a window that's been split with `split_window()`. Owns that
window/region's 3-d and 2-d scene graph roots, camera(s), trackball, mouse
and keyboard wiring, model loading helpers, rendering-mode toggles, and the
animation-scrubber UI created by `next_anim_control()`. Constructors are
`protected` — only `PandaFramework` (a `friend`) and `split_window()` create
instances.

## Behavior notes

- **Two constructors, two different jobs.** The "new window" ctor
  (`WindowFramework(PandaFramework*)`) just zeroes flags; the actual
  `GraphicsOutput`/camera/display-region setup happens in the separate
  `open_window()` method, called by `PandaFramework::open_window()`. The
  "split" ctor (`WindowFramework(const WindowFramework&, DisplayRegion*)`)
  copies the *same* `_window` pointer from an existing `WindowFramework`,
  takes ownership of a new `DisplayRegion` on that same output, and
  immediately creates its own camera (`make_camera()`) bound to that region
  — so a split window shares the `GraphicsOutput` but gets an independent
  camera, scene graph roots (lazily, on first use), trackball, and mouse.
- **`open_window()` (protected, the real one) always creates a fresh 3-d
  `DisplayRegion` covering the whole window**, and explicitly disables the
  window's own clear flags (`set_clear_color/depth/stencil_active(false)`)
  in favor of the `DisplayRegion`'s — this is what lets multiple
  `DisplayRegion`s (from splitting) each clear to a different background
  color. It also unconditionally calls `make_camera()` and
  `set_background_type(_background_type)`.
- **`get_render()` sets two-sided off and installs default
  rescale-normal/smooth-shading attribs** the first time it's called —
  every `WindowFramework`'s 3-d scene starts from these defaults, not
  Panda's global defaults.
- **`get_render_2d()`'s ortho camera uses a fixed `(-1,1)` film size** in
  both axes regardless of window shape (see [README's 2-d node
  hierarchy](README.md#shared-concepts) for how `aspect_2d`/`pixel_2d`
  correct for that). `get_aspect_2d()` additionally wires PGUI mouse
  support as a side effect: it calls `get_mouse()` and, if that node is a
  `MouseWatcher`, hands it to the new `PGTop` via `set_mouse_watcher()`
  (`get_pixel_2d()` does not) — skip `get_aspect_2d()` and PGUI widgets
  parented under `render_2d` receive no mouse events.
- **`get_mouse()` conditionally constrains itself to a `DisplayRegion`** —
  only when `_window->get_side_by_side_stereo()` is true does the
  `MouseWatcher` get `set_display_region()` called with the window's overlay
  region, so split-stereo setups track mouse position per-half. Non-stereo
  windows get an unconstrained `MouseWatcher`.
- **`enable_keyboard()` and `setup_trackball()` are both no-ops on a window
  with zero input devices** (checked via
  `DCAST(GraphicsWindow, _window)->get_num_input_devices() > 0`), which is
  the normal case for an offscreen buffer — calling them on a buffer-backed
  `WindowFramework` silently does nothing (and `_got_keyboard`/`_got_trackball`
  are still marked true, so a later call is *also* a no-op, not a retry).
- **`center_trackball()` derives view distance from the FOV of the first
  camera with a lens**, not a fixed multiplier: `distance = radius /
  tan(min(fov.x, fov.y)/2)`, and also pushes the lens's far plane out and
  near plane in to guarantee the whole bounding sphere is inside the
  frustum. If the object's bounds are infinite or empty, it logs a warning
  and returns without moving the trackball.
- **`load_model()` decides whether to search the model path** based on
  whether the filename is already fully qualified or already exists as
  given (`vfs->exists(filename)`) — otherwise it sets `LoaderOptions::LF_search`
  so `model-path`-relative names resolve. It also detects image files by
  extension (via `TexturePool`'s known-extension list) and routes those
  through `load_image_as_model()` instead of the regular model loader,
  turning a bare image into a textured card (or a textured cube, for
  3-d/cube-map textures).
- **`load_default_model()` and `get_shuttle_controls_font()` decode
  compiled-in asset data**, not files on disk — the rock-floor texture comes
  from `rock_floor.rgb_src.c` via a `PNMImage::read()` on an in-memory
  stream, and the shuttle-controls font comes from `shuttle_controls.bam_src.c`
  via `BamFile::open_read()` on an in-memory stream. `get_shuttle_controls_font()`
  caches its result in the **static, class-wide** `_shuttle_controls_font` —
  shared across all `WindowFramework` instances, loaded once per process.
- **`next_anim_control()` state machine**: disabled → enabled at anim index
  0 (auto-plays if any anim exists) → each call while enabled advances to
  the next anim and plays it → advancing past the last anim disables
  controls again and calls `_anim_controls.loop_all(true)` to resume looping
  everything. `set_anim_controls(false)` mid-sequence just tears down the UI
  without resuming looping.
- **`create_anim_controls()` always calls `destroy_anim_controls()` first**
  — safe to call repeatedly; it's how `next_anim_control()` rebuilds the UI
  for each new anim. The controls UI itself is built with `PGItem`/`PGButton`
  /`PGSliderBar` under `aspect_2d`; sliders/buttons carry
  `MouseWatcherRegion::SF_mouse_button` suppress flags so clicks on them
  don't also fall through to the 3-d scene. `destroy_anim_controls()` removes **all** event
  hooks tagged with `(void*)this` (`remove_hooks_with`), which drops the
  jog-shuttle button hooks registered in `setup_shuttle_button()`.
- **`update_anim_controls()` is a live per-frame task**
  (`st_update_anim_controls`, added in `create_anim_controls()`) — while the
  slider thumb is held down it poses the anim to the slider's value instead
  of letting it play, giving scrub behavior; otherwise it just reflects the
  anim's current frame back onto the slider and label.
- **`adjust_dimensions()` picks film-size vs. aspect-ratio strategy per
  camera lens depending on whether the window reports a nonzero `y_size`** —
  with a known pixel size it sets an explicit film size (matches pixel
  aspect exactly); without one (e.g. some offscreen/headless cases) it falls
  back to `set_aspect_ratio()`. This is called from
  `PandaFramework::event_window_event` on resize, not automatically by
  `WindowFramework` itself.
- **`split_window()` never recombines** — the header/cxx comment says so
  explicitly; there's no public API to undo a split. It picks
  horizontal/vertical automatically in `ST_default` mode based on which
  pixel dimension of the current `DisplayRegion` is larger, then shrinks the
  existing region and creates a new sibling region (and a new
  `WindowFramework` for it) covering the other half. The 2-d `DisplayRegion`
  is resized to match only if it already exists (`_display_region_2d !=
  nullptr`) — if `get_render_2d()` was never called before splitting, the
  new half gets no matching 2-d region adjustment.
- **Rendering-toggle setters interact with each other.** `set_wireframe()`
  clears two-sidedness first unless `_two_sided_enabled` is already set, and
  filled-wireframe mode additionally darkens the scene via
  `set_color_scale()` so wireframe remains visible against a white scene.
  `set_two_sided(true)` and `set_one_sided_reverse(true)` are mutually
  exclusive — each setter unconditionally clears the other's flag
  (`_one_sided_reverse_enabled = false` inside `set_two_sided`, and vice
  versa) even if wireframe mode means the attrib change itself is
  suppressed. `override_priority` (100) is used on every one of these
  attribs so per-window toggles win over whatever a loaded model's own
  render state specifies.
- **`set_lighting(true)` lazily creates lights via `setup_lights()`** only
  once (`_got_lights` guard) — toggling lighting off and back on reuses the
  same ambient/directional light nodes rather than recreating them.

## API

### Construction / lifetime (protected — see behavior notes)

| Signature | Notes |
|---|---|
| `WindowFramework(PandaFramework *panda_framework)` | Used for a fresh window. |
| `WindowFramework(const WindowFramework &copy, DisplayRegion *display_region)` | Used by `split_window()`. |
| `virtual ~WindowFramework()` | Calls `close_window()`. |
| `GraphicsOutput *open_window(...)` (protected) | Creates the `GraphicsOutput`, 3-d `DisplayRegion`, camera, optional frame-rate/scene-graph-analyzer meters. |
| `void close_window()` (protected) | Tears down window, scene roots, mouse, lights, meters; resets rendering flags (note: `_texture_enabled` resets to `true`, matching the constructor default). |

### Accessors

| Signature | Notes |
|---|---|
| `INLINE PandaFramework *get_panda_framework() const` | |
| `INLINE GraphicsWindow *get_graphics_window() const` | `nullptr` if the output isn't a `GraphicsWindow` (e.g. an offscreen buffer). |
| `INLINE GraphicsOutput *get_graphics_output() const` | |
| `NodePath get_camera_group()` | Lazily created under `get_render()`. |
| `INLINE int get_num_cameras() const` / `INLINE Camera *get_camera(int n) const` | |
| `INLINE DisplayRegion *get_display_region_2d() const` / `INLINE DisplayRegion *get_display_region_3d() const` | |
| `NodePath get_render()` | Lazy; sets rescale-normal + smooth-shade + two-sided-off defaults. |
| `NodePath get_render_2d()` | Lazy; creates the 2-d ortho camera + `DisplayRegion` (sort 10). |
| `NodePath get_aspect_2d()` | Lazy; `PGTop` under `render_2d`, aspect-corrected, wired to the `MouseWatcher`. |
| `NodePath get_pixel_2d()` | Lazy; `PGTop` under `render_2d`, pixel-unit coordinates, upper-left origin. |
| `NodePath get_mouse()` | Lazy; `MouseWatcher` under the framework's shared per-window mouse node. |
| `NodePath get_button_thrower()` | Empty until `enable_keyboard()` has been called. |
| `INLINE bool get_anim_controls() const` | |
| `static TextFont *get_shuttle_controls_font()` | Process-wide cached, decoded from compiled-in bam data. |

### Input setup

| Signature | Notes |
|---|---|
| `void enable_keyboard()` | Idempotent; no-op if the output has no input devices. Attaches a `ButtonThrower` tagged with `EventParameter(this)`. |
| `void setup_trackball()` | Idempotent; no-op if no input devices. Places the trackball 50 units back by default. |
| `void center_trackball(const NodePath &object)` | See behavior notes for the FOV-based distance calc. |

### Model loading

| Signature | Notes |
|---|---|
| `bool load_models(const NodePath &parent, int argc, char *argv[], int first_arg = 1)` | Convenience wrapper over the filename-vector overload. |
| `bool load_models(const NodePath &parent, const pvector<Filename> &files)` | Returns false if *any* model failed to load (others still get attached). |
| `NodePath load_model(const NodePath &parent, Filename filename)` | Returns `NodePath::not_found()` on failure. Also handles bare image files — see behavior notes. |
| `NodePath load_default_model(const NodePath &parent)` | Textured blue triangle, for "something to look at" when no model is given. |

### Animation

| Signature | Notes |
|---|---|
| `void loop_animations(int hierarchy_match_flags = PartGroup::HMF_ok_part_extra \| PartGroup::HMF_ok_anim_extra)` | Calls global `auto_bind()` against `get_render()`, then `loop_all(true)`. |
| `void stagger_animations()` | Randomizes each bound anim's play rate to `[0.9, 1.1)` so parallel characters desync. |
| `void next_anim_control()` | Cycles the onscreen anim-scrubber through bound anims. See behavior notes. |
| `void set_anim_controls(bool enable)` | Directly show/hide the scrubber UI. |

### Window/region geometry

| Signature | Notes |
|---|---|
| `void adjust_dimensions()` | Re-derives aspect2d scale, pixel2d scale, and lens film size/aspect from the window's current pixel size. Call after a resize. |
| `enum BackgroundType { BT_other, BT_default, BT_black, BT_gray, BT_white, BT_none }` | `BT_other` means "app manages clearing itself" (setter is a no-op). |
| `enum SplitType { ST_default, ST_horizontal, ST_vertical }` | |
| `WindowFramework *split_window(SplitType split_type = ST_default)` | See behavior notes — never recombinable. |

### Rendering toggles

| Signature |
|---|
| `void set_wireframe(bool enable, bool filled=false)` / `INLINE bool get_wireframe() const` / `INLINE bool get_wireframe_filled() const` |
| `void set_texture(bool enable)` / `INLINE bool get_texture() const` |
| `void set_two_sided(bool enable)` / `INLINE bool get_two_sided() const` |
| `void set_one_sided_reverse(bool enable)` / `INLINE bool get_one_sided_reverse() const` |
| `void set_lighting(bool enable)` / `INLINE bool get_lighting() const` |
| `void set_perpixel(bool enable)` / `INLINE bool get_perpixel() const` |
| `void set_background_type(BackgroundType type)` / `INLINE BackgroundType get_background_type() const` |

### Camera

| Signature | Notes |
|---|---|
| `NodePath make_camera()` | Adds a new `Camera` + `PerspectiveLens` under `get_camera_group()`, appends to `_cameras`. Called automatically by `open_window()`/split ctor for the default camera; can be called again for additional cameras. |

## See also

- [PandaFramework.md](PandaFramework.md) — owns the list of `WindowFramework`s and the shared mouse/data-root/task manager.
- [README.md](README.md) — 2-d node hierarchy, config variables, mouse wiring.
- [../pgui/PGSliderBar.md](../pgui/PGSliderBar.md), [../pgui/PGButton.md](../pgui/PGButton.md) — used by the anim-control scrubber UI.
- [../event/AsyncTaskManager.md](../event/AsyncTaskManager.md) — hosts the `st_update_anim_controls` per-frame task.
