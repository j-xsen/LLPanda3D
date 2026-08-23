# CullTraverser

**Source:** `panda/src/pgraph/cullTraverser.h` (+ `.I`, `.cxx`)
**Inherits:** TypedReferenceCount

The engine's depth-first scene-graph walker for one frame: descends from a root `NodePath`, accumulates `RenderState`/transform, performs view-frustum culling (plus clip-plane/portal/occluder culling via [CullPlanes](CullPlanes.md)), and for every visible Geom found in a [GeomNode](GeomNode.md), wraps it as a [CullableObject](CullableObject.md) and hands it to a [CullHandler](CullHandler.md) (in practice, a [CullResult](CullResult.md)). This is the central algorithm of the whole cull pipeline — everything else in this group ([CullTraverserData](CullTraverserData.md), [CullPlanes](CullPlanes.md), [PortalClipper](PortalClipper.md)) exists to support it.

## Behavior notes

- **Setup is two-phase:** construct, then call `set_scene()` with a [SceneSetup](SceneSetup.md) and GSG before `traverse()` — `set_scene()` pulls `_initial_state`, camera tag-state key, and camera mask off the `SceneSetup`/`Camera`, and computes `_effective_incomplete_render` as `gsg->get_incomplete_render() && dr_incomplete_render` (both the GSG-wide and the per-`DisplayRegion` incomplete-render flags must agree).
- **Portal culling branch:** if `allow-portal-cull` is set, `traverse(const NodePath&)` does something unusual — it runs the traversal **twice**. First pass constructs a [PortalClipper](PortalClipper.md) seeded from the lens frustum and traverses normally (this pass discovers portals and clips the frustum through them, accumulating clipped sub-frustums in `portal_viewer._previous`). Then it re-runs traversal a second time using the clipped result (`my_data`, built from `portal_viewer._previous`), transformed relative to the scene's cull-center. Without portal culling, it's a single top-level `do_traverse()` call.
- **`do_traverse()` (inlined for recursion perf)** is the actual per-node visit: test `is_in_view()`; if visible and the node has anything "fancy" (compiled bits like a cull callback, or active `_cull_planes`), apply the node's transform/state via `CullTraverserData::apply_transform_and_state()`, adjust any `FogAttrib`'s fog to the camera (`Fog::adjust_to_camera()`), and if the node has `PandaNode::FB_cull_callback` set, invoke `node->cull_callback()` — a false return here prunes the subtree (the node opted out). Otherwise it falls through to `traverse_below()`.
- **`traverse_below()`** does the actual work at a node: unless `data.is_this_node_hidden(camera_mask)`, calls `node->add_for_draw()` (the node's chance to emit its own Geoms into the traverser) and, if the node has a `DecalEffect`, composes a `DepthOffsetAttrib` into `_state` so every descendant gets offset — this is called out explicitly as **the only mechanism decals are implemented with** (`get_depth_offset_decals()` always returns `true`). It then recurses into children — via `get_first_visible_child()`/`get_next_visible_child()` instead of a plain loop if `node->has_selective_visibility()` (e.g. `SwitchNode`/`SequenceNode`-style nodes that only show one child at a time; those live in `pgraphnodes`, undocumented here).
- **Bounds visualization** (`show_bounds()`/`make_bounds_viz()`/`make_tight_bounds_viz()`) is a large chunk of this file but purely a debug feature, triggered per-node by `RenderEffects::has_show_bounds()` (see `ShowBoundsEffect`). It builds throwaway wireframe/solid Geom visualizations of a node's bounding sphere/box/hexahedron (or, for "tight" bounds, the actual computed tight AABB via `calc_tight_bounds()`) and injects them as extra `CullableObject`s with fixed debug `RenderState`s (green-ish flat-colored wireframe, front/back-face split into "outer"/"inner" states for correct depth perception).
- `is_in_view()` just forwards to `CullTraverserData::is_in_view()` — kept as a separate overridable virtual so a subclass can customize visibility logic without touching frustum-test internals.
- `flush_level()` flushes four static `PStatCollector`s (`Nodes`, `Nodes:GeomNodes`, `Geoms`, `Geoms:Occluded`) used for profiling — call once per frame after traversal, not per-node.
- The copy constructor exists specifically to let `PortalClipper`-driven code spin up a second, independent traversal sharing the same GSG/scene/handler/frustum settings.

## API

| Signature | Notes |
|---|---|
| `CullTraverser()` | |
| `GraphicsStateGuardianBase *get_gsg() const` / `Thread *get_current_thread() const` | |
| `void set_scene(SceneSetup*, GraphicsStateGuardianBase*, bool dr_incomplete_render)` | must be called before `traverse()` |
| `SceneSetup *get_scene() const` | |
| `bool has_tag_state_key() const` / `const std::string &get_tag_state_key() const` | camera's per-node tag-state override key, see [Camera](Camera.md) |
| `void set_camera_mask(DrawMask)` / `get_camera_mask() const` | which draw-mask bits are visible; normally set from the camera |
| `const TransformState *get_camera_transform() const` / `get_world_transform() const` | forwarded from the `SceneSetup` |
| `const RenderState *get_initial_state() const` | state applied as if set at the traversal root |
| `bool get_depth_offset_decals() const` | always `true` |
| `void set_view_frustum(GeometricBoundingVolume*)` / `get_view_frustum() const` | root-space frustum; `nullptr` disables frustum culling |
| `void set_cull_handler(CullHandler*)` / `get_cull_handler() const` | must be set before `traverse()` |
| `void set_portal_clipper(PortalClipper*)` / `get_portal_clipper() const` | |
| `bool get_effective_incomplete_render() const` | |
| `void traverse(const NodePath &root)` | entry point; handles the portal-culling double-pass internally |
| `void traverse(CullTraverserData &data)` | continue traversal from an already-built `CullTraverserData` |
| `virtual void traverse_below(CullTraverserData &data)` | visits one node's children; overridable |
| `virtual void end_traverse()` | calls `cull_handler->end_traverse()`; call once traversal is done |
| `static void flush_level()` | flush profiling collectors |
| `void draw_bounding_volume(const BoundingVolume*, const TransformState *internal_transform) const` | debug viz helper |
| `virtual bool is_in_view(CullTraverserData &data)` | protected/overridable; default forwards to `CullTraverserData::is_in_view()` |

## Usage

```cpp
CullTraverser trav;
trav.set_scene(scene_setup, gsg, dr->get_incomplete_render());
trav.set_cull_handler(cull_result);   // a CullHandler, e.g. CullResult
trav.set_view_frustum(view_frustum);  // or nullptr to disable frustum culling
trav.traverse(scene_root);
trav.end_traverse();
```

## See also

- [CullTraverserData](CullTraverserData.md) — per-node state threaded through the recursion
- [CullPlanes](CullPlanes.md), [PortalClipper](PortalClipper.md) — clip-plane/portal/occluder culling support
- [CullHandler](CullHandler.md), [CullResult](CullResult.md) — receives the culled [CullableObject](CullableObject.md)s
- [SceneSetup](SceneSetup.md) — camera/lens/viewport snapshot consumed by `set_scene()`
