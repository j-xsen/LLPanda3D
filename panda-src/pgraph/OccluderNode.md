# OccluderNode

**Source:** `panda/src/pgraph/occluderNode.h` (+ `.I`, `.cxx`)
**Inherits:** PandaNode

Holds a single rectangular occluder polygon (4 vertices, counterclockwise facing). Once activated on a camera (e.g. `camera_node.node()->set_occluder(occluder_node_path)` — see [Camera](Camera.md)), objects whose bounding volume falls entirely behind the occluder from the camera's viewpoint are culled — see [CullPlanes::apply_state()](CullPlanes.md#behavior-notes) for the actual occluder-volume construction and rejection logic run during traversal.

## Behavior notes

- **Hidden by default:** the constructor calls `set_overall_hidden(true)` — an `OccluderNode` never renders its own geometry in normal operation; it exists purely as culling data consumed elsewhere ([CullPlanes](CullPlanes.md)).
- **Debug visualization is the entire content of `cull_callback()`:** since the node is normally hidden, `cull_callback()` only runs when something has explicitly un-hidden it (e.g. `show_occluder_volumes` debug tooling). When it does run, it unconditionally records two `CullableObject`s — a checkerboard-textured polygon (`get_occluder_viz()`/`get_occluder_viz_state()`) and a wireframe frame outline (`_frame_viz`/`get_frame_viz_state()`) — directly into the traverser's `CullHandler`, bypassing the normal Geom-based draw path entirely (this node holds no `GeomNode`-style renderable Geoms of its own).
- `set_double_sided(true)` makes the *back* face also occlude — otherwise (default) an occluder viewed/approached from behind is ignored (see [CullPlanes](CullPlanes.md#behavior-notes)'s back-facing rejection and its double-sided special case that flips vertex winding instead of rejecting).
- `set_min_coverage(value)` (0.0–1.0) is a performance/quality tradeoff: an occluder that doesn't cover at least this fraction of the screen is skipped, since the CPU cost of accumulating and testing against a small occluder volume may exceed the fill-rate it saves. Default is `0.0` (no minimum).
- Default vertices form a 2×2 unit square in the XZ plane (`LPoint3::rfu(±1, 0, ±1)`), same convention as `PortalNode`'s default square.
- `_viz_tex` is a single shared static `Texture` (the checkerboard pattern) used by every `OccluderNode`'s debug visualization — loaded/created once, not per-instance.

## API

| Signature | Notes |
|---|---|
| `OccluderNode(const std::string &name)` | |
| `void set_double_sided(bool)` / `is_double_sided()` | default false |
| `void set_min_coverage(PN_stdfloat)` / `get_min_coverage()` | default 0.0, range 0–1 |
| `void set_vertices(v0, v1, v2, v3)` | replaces all 4 vertices at once |
| `size_t get_num_vertices() const` / `const LPoint3 &get_vertex(size_t) const` / `set_vertex(size_t, LPoint3)` / `get_vertices()` (MAKE_SEQ) | always 4 |
| `virtual bool cull_callback(CullTraverser*, CullTraverserData&)` | debug-viz only, see behavior notes |

## Usage

```cpp
NodePath render("render");
PT(OccluderNode) occluder = new OccluderNode("wall");
occluder->set_vertices(LPoint3(-5,0,-3), LPoint3(5,0,-3),
                        LPoint3(5,0,3), LPoint3(-5,0,3));
NodePath occluder_np = render.attach_new_node(occluder);
camera_np.node()->set_occluder(occluder_np);  // Camera, not this class
```

## See also

- [CullPlanes](CullPlanes.md) — consumes occluders via `OccluderEffect` during traversal, does the actual visibility math
- [OccluderEffect](OccluderEffect.md) — the `RenderEffect` that activates an occluder on a subtree
- [Camera](Camera.md) — `set_occluder()`/`clear_occluder()` live here
- [PortalNode](PortalNode.md) — the other polygon-based visibility-culling node type in this module
