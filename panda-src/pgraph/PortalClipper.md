# PortalClipper

**Source:** `panda/src/pgraph/portalClipper.h` (+ `.I`, `.cxx`)
**Inherits:** TypedObject

Implements portal-based visibility culling: given a camera frustum and one [PortalNode](PortalNode.md) at a time, clips the current frustum through the portal's polygon to produce a tighter "reduced frustum" that's passed on to the cell the portal leads into. One `PortalClipper` is created per [CullTraverser](CullTraverser.md)::traverse() call when `allow-portal-cull` is enabled (see [CullTraverser](CullTraverser.md#behavior-notes)'s two-pass portal traversal), and is threaded through the traversal via `CullTraverser::set_portal_clipper()`/`get_portal_clipper()`.

## Behavior notes

- **Driven externally by `PortalNode::cull_callback()`**, not by `CullTraverser` directly: when the traverser visits a `PortalNode`, the node's own `cull_callback()` calls `prepare_portal(node_path)` on the traverser's active `PortalClipper`. If that returns true and yields a non-null `get_reduced_frustum()`, the portal node then manually calls `trav->traverse_below()` on a modified `CullTraverserData` (transformed into the target cell's space, with `_portal_depth` incremented) — the portal doesn't just narrow the frustum, it's the thing that literally continues the recursion into the connected cell.
- **`is_facing_view(plane)`** and **`is_whole_portal_in_view(cmat)`** are cheap early-rejection tests `prepare_portal()` runs before doing the expensive frustum-clipping math: a portal facing away from the camera (`portal_plane[3] <= 0`, in camera space) or entirely outside the current reduced frustum is skipped without computing a new hexahedron.
- **State stacking:** `_reduced_frustum`, `_reduced_viewport_min/max`, and `_clip_state` are mutated in place as the traverser descends into a portal, and `PortalNode::cull_callback()` explicitly saves and restores them (via `get_reduced_frustum()`/`set_reduced_frustum()` etc.) around the recursive `traverse_below()` call — this is how sibling portals at the same graph level each see the *parent* portal's reduced frustum rather than accumulating each other's.
- `_previous` is a `GeomNode` accumulating debug-visualization line/point geometry (frustum wireframes, portal boundaries) built up via the `move_to()`/`draw_to()` pen-style API and flushed with `draw_lines()`/`draw_camera_frustum()`; it's also (per [CullTraverser](CullTraverser.md#behavior-notes)) reused as the geometry base for the *second* portal-culling traversal pass. Only populated when `debug-portal-cull` is on.
- `_view_frustum` is set once at construction (the camera's original, unclipped frustum) and never mutated afterward; `_reduced_frustum` starts equal to it and gets progressively narrowed as `prepare_portal()` is called for each portal along a chain of nested cells.

## API

| Signature | Notes |
|---|---|
| `PortalClipper(GeometricBoundingVolume *frustum, SceneSetup *scene_setup)` | constructed once per top-level `CullTraverser::traverse()` call |
| `bool prepare_portal(const NodePath &node_path)` | the core operation: clips the current reduced frustum through the portal at `node_path`; on success, `get_reduced_frustum()` returns the result |
| `BoundingHexahedron *get_reduced_frustum() const` / `set_reduced_frustum(...)` | current clipped frustum, camera space |
| `void get_reduced_viewport(LPoint2 &min, LPoint2 &max) const` / `set_reduced_viewport(...)` | screen-space bounds of the reduced frustum |
| `const RenderState *get_clip_state() const` / `set_clip_state(...)` | remembers a portal's clip-plane state for its children |
| `bool is_facing_view(const LPlane &portal_plane)` | cheap facing test |
| `bool is_whole_portal_in_view(const LMatrix4 &cmat)` | cheap containment test |
| `void move_to(x, y, z)` / `draw_to(x, y, z)` | pen-style debug line building |
| `void draw_hexahedron(BoundingHexahedron*)` / `draw_camera_frustum()` | queues a wireframe hexahedron for debug drawing |
| `void draw_lines()` | flushes queued debug lines into `_previous` |
| `void draw_current_portal()` | debug-draws the portal currently being processed |
| `PT(GeomNode) _previous` | public field, accumulated debug geometry (also reused as pass-2 traversal seed) |
| `SceneSetup *_scene_setup` | public field |

## Usage

Not constructed by application code — created internally by [CullTraverser::traverse()](CullTraverser.md) when `allow-portal-cull` is on, and driven by [PortalNode::cull_callback()](PortalNode.md) as the traversal encounters portals. Application code's involvement is limited to placing `PortalNode`s in the scene graph and setting `allow-portal-cull`/`debug-portal-cull`.

## See also

- [PortalNode](PortalNode.md) — drives calls into this class during traversal
- [CullTraverser](CullTraverser.md) — owns the `PortalClipper` instance and runs the two-pass portal traversal
- [CullTraverserData](CullTraverserData.md) — carries `_portal_depth`, checked against `PortalNode::get_max_depth()`
