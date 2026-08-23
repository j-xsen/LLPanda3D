# SceneSetup

**Source:** `panda/src/pgraph/sceneSetup.h` (+ `.I`, `.cxx`)
**Inherits:** TypedReferenceCount

A plain-data snapshot of everything needed to render one scene through one camera for one frame: the [Camera](Camera.md)/Lens (`panda/src/mathutil`, undocumented), the scene root and camera `NodePath`s, the viewport size, the initial `RenderState`, and a set of derived `TransformState`s (camera↔world, and their GSG-internal-coordinate-system equivalents). Built once per `DisplayRegion` per frame (by the display layer, before culling begins) and read throughout the [CullTraverser](CullTraverser.md) walk via `CullTraverser::get_scene()`.

## Behavior notes

- Almost entirely a getter/setter bag — the only non-trivial logic is `get_cull_center()` and `get_cull_bounds()`:
  - `get_cull_center()` returns `Camera::get_cull_center()` if the camera has one set (lets you cull as if viewed from a different point than the actual camera, e.g. a wider "cull frustum" for temporal effects), falling back to the actual `_camera_path`.
  - `get_cull_bounds()` similarly prefers `Camera::get_cull_bounds()` if overridden, otherwise derives bounds from the `Lens` (`_lens->make_bounds()`).
- The default constructor pre-fills `_initial_state` to `RenderState::make_empty()` and all four transform fields to `TransformState::make_identity()`, so a freshly-constructed `SceneSetup` is safe to query before every field is explicitly set.
- `_camera_transform`/`_world_transform` are inverses of each other (camera-relative-to-root vs. root-relative-to-camera); `_cs_transform`/`_cs_world_transform` are the same pair further composed with the transform into the GSG's internal coordinate system (left- vs. right-handed, Y-up vs. Z-up, as configured per-GSG).

## API

| Signature | Notes |
|---|---|
| `DisplayRegion *get_display_region() const` / `set_display_region(...)` | |
| `int get_viewport_width/height() const` / `set_viewport_size(w, h)` | pixels |
| `const NodePath &get_scene_root() const` / `set_scene_root(...)` | |
| `const NodePath &get_camera_path() const` / `set_camera_path(...)` | |
| `Camera *get_camera_node() const` / `set_camera_node(...)` | |
| `const Lens *get_lens() const` / `set_lens(...)` | |
| `bool get_inverted() const` / `set_inverted(bool)` | mirror-flip rendering, for reflection/portal effects |
| `const NodePath &get_cull_center() const` | camera, or `Camera::get_cull_center()` override |
| `PT(BoundingVolume) get_cull_bounds() const` | lens bounds, or `Camera::get_cull_bounds()` override |
| `const RenderState *get_initial_state() const` / `set_initial_state(...)` | applied as if set at the scene root |
| `const TransformState *get_camera_transform() const` / `set_camera_transform(...)` | camera relative to scene root |
| `const TransformState *get_world_transform() const` / `set_world_transform(...)` | inverse of camera_transform |
| `const TransformState *get_cs_transform() const` / `set_cs_transform(...)` | camera-space → GSG internal space |
| `const TransformState *get_cs_world_transform() const` / `set_cs_world_transform(...)` | world_transform composed into GSG internal space |

## Usage

Application code rarely constructs a `SceneSetup` directly — it's built by the display/rendering layer per `DisplayRegion` per frame. Read access inside a custom cull-related callback:

```cpp
SceneSetup *scene = traverser->get_scene();
const Lens *lens = scene->get_lens();
NodePath cull_center = scene->get_cull_center();
```

## See also

- [CullTraverser](CullTraverser.md) — reads this once per traversal via `get_scene()`
- [Camera](Camera.md) — source of cull-center/cull-bounds overrides
- [display](../display/README.md) — `DisplayRegion`/`GraphicsEngine`, which construct `SceneSetup` each frame
