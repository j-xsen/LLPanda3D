# Camera

**Source:** `panda/src/pgraph/camera.h` (+ `.I`, `.cxx`)
**Inherits:** [LensNode](LensNode.md) > PandaNode

A node that can be positioned around in the scene graph to represent a
point of view for rendering a scene. A `Camera` is typically parented into
the scene it renders (as of the modern API); one or more
[`DisplayRegion`](../display/DisplayRegion.md)s reference a `Camera` to
say what to draw into that region of a window.

## Behavior notes

- `_camera_mask` defaults to `~PandaNode::get_overall_bit()` (everything
  except the reserved "overall" bit). During cull, a node is skipped
  unless its draw mask intersects the camera's mask — this is the
  mechanism for hiding parts of the scene graph from specific cameras.
  Using Panda's reserved "overall" bit as a camera mask is asserted
  against in `set_camera_mask()`.
- `set_scene()` is **deprecated**: without an explicit scene set, a Camera
  renders whatever scene graph it's parented into. This is the modern,
  preferred mechanism — `_scene` exists mainly for legacy code.
- `set_cull_center()`/`set_cull_bounds()` let culling be performed from a
  different viewpoint/volume than the camera itself (e.g. for debugging
  the culling process by moving the camera away while culling stays fixed
  at the old position).
- `set_lod_center()` independently offsets where LOD distance is measured
  from — useful to avoid LOD popping when the camera orbits an avatar in
  a small circle; `set_lod_scale()` is a multiplier combined with each
  `LodNode`'s own scale.
- `set_initial_state()` applies a `RenderState` as if it were set at the
  very top of the scene graph for this camera only — lets one camera see
  the scene with a global override (e.g. wireframe) without touching the
  actual graph.
- **Tag states**: `set_tag_state_key()` names a tag; when a node
  encountered during traversal has that tag with a value matching one
  registered via `set_tag_state()`, this camera applies the associated
  `RenderState` to that node — but *only* when rendered by this camera.
  Used for multipass rendering where a specialty camera needs different
  effects on tagged nodes (e.g. a shadow-caster pass).
- `AuxSceneData` objects can be stashed per-`NodePath` on the camera
  (`set_aux_scene_data()`) as an association cache; `cleanup_aux_scene_data()`
  sweeps out entries past their `get_expiration_time()`, driven by the
  global clock.
- `safe_to_flatten()` and `safe_to_transform()` both return `false` — the
  `Camera` pointer itself is meaningful (referenced by DisplayRegions), so
  it must never be duplicated or have its transform baked away.
- The destructor asserts `_display_regions` is empty — DisplayRegions are
  expected to detach themselves from the Camera before it's destroyed.
  `add_display_region()`/`remove_display_region()` are private, called
  only by `DisplayRegion` (a `friend class`).

## API

| Method | Notes |
|---|---|
| `Camera(name, Lens *lens = new PerspectiveLens())` | Constructor; lens defaults to a fresh `PerspectiveLens` |
| `set_active(bool)` / `is_active()` | When inactive, nothing is rendered by this camera |
| `set_scene(NodePath)` / `get_scene()` | Deprecated explicit scene root override |
| `get_num_display_regions()` / `get_display_region(n)` | Read-only view of associated DisplayRegions |
| `set_camera_mask(DrawMask)` / `get_camera_mask()` | Bits controlling which scene-graph subset is visible to this camera |
| `set_cull_center(NodePath)` / `get_cull_center()` | Alternate viewpoint for culling |
| `set_cull_bounds(BoundingVolume*)` / `get_cull_bounds()` | Override the frustum bounding volume used for culling |
| `set_lod_center(NodePath)` / `get_lod_center()` | Alternate viewpoint for LOD distance measurement |
| `set_lod_scale(float)` / `get_lod_scale()` | Global LOD distance multiplier |
| `set_initial_state(RenderState*)` / `get_initial_state()` | State applied as if at the scene root, for this camera only |
| `set_tag_state_key(string)` / `get_tag_state_key()` | Tag name this camera watches for per-node state overrides |
| `set_tag_state(tag_value, RenderState*)` / `clear_tag_state()` / `clear_tag_states()` / `has_tag_state()` / `get_tag_state()` | Register/query tag-value → state associations |
| `set_aux_scene_data(NodePath, AuxSceneData*)` / `clear_aux_scene_data()` / `get_aux_scene_data()` / `list_aux_scene_data(ostream&)` / `cleanup_aux_scene_data()` | Per-NodePath auxiliary data cache with expiration sweep |

## Usage

```cpp
NodePath camera_np = window->get_camera_group();
// or, when building manually:
PT(Camera) camera = new Camera("cam");
camera->set_lens(new PerspectiveLens());
NodePath cam_np = render.attach_new_node(camera);

DisplayRegion *dr = window->make_display_region();
dr->set_camera(cam_np);
```

## See also

- [LensNode](LensNode.md) — base class; holds the `Lens` this Camera uses
- [`display/DisplayRegion`](../display/DisplayRegion.md) — references a
  Camera to render into a window region
- [RenderState](RenderState.md) — used by `initial_state`/tag states
