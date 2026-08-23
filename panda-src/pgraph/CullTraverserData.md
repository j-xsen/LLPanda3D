# CullTraverserData

**Source:** `panda/src/pgraph/cullTraverserData.h` (+ `.I`, `.cxx`)

The per-node state bundle threaded through [CullTraverser](CullTraverser.md)'s recursive descent: which node, its net transform and accumulated `RenderState` so far, the current view frustum (in this node's local space), and the active [CullPlanes](CullPlanes.md). Passed by mutable reference down the recursion — the traverser mutates fields of a single `CullTraverserData` in place as it walks into each node, rather than the caller building a new immutable object at every level, for performance (this is the "collects together pieces of data accumulated for each node" mentioned in the class comment; existing separately from `CullTraverser` itself mainly to keep `CullTraverser`'s own method signatures short).

## Behavior notes

- **Two constructors, two roles:** the root constructor (`start` NodePath, initial transform/state/frustum) begins a traversal; the child constructor `CullTraverserData(parent, child_node)` derives a new instance from a parent for stepping into one child — it does **not** copy the parent's `_net_transform`/`_state` verbatim, those get updated by `apply_transform_and_state()` once the traverser decides to descend.
- **NodePath is reconstructed lazily and only on demand.** Rather than carrying a full `NodePath` at every recursion level (expensive), each level stores just a `PandaNode*` chain (`_next` pointing at the parent `CullTraverserData`, `_start` only set at the root). `get_node_path()`/`r_get_node_path()` walks this chain and calls `PandaNode::get_component()` to reconstruct a proper `NodePathComponent` chain only when actually asked for — e.g. for debug-spam logging or building a `CullPlanes` key. If the ancestry chain turns out to be disconnected (a node was removed from its parent mid-traversal), it silently truncates the `NodePath` at the break rather than asserting.
- **`apply_transform_and_state()`** is where a node's own contribution gets folded into the accumulator:
  1. Starts from the node's own `RenderState` (`_node_reader.get_state()`), and if the traverser has a *tag state key* set and this node is tagged with it, composes in `Camera::get_tag_state(tag)` — this is the mechanism behind per-camera state overrides (e.g. render-to-texture passes that want a different shader on tagged nodes).
  2. Composes the node's draw mask into `_draw_mask` (for `is_this_node_hidden()` bit-mask culling).
  3. Applies the node's transform — **unless** `RenderEffects::has_cull_callback()`, in which case the effect (e.g. [CompassEffect](CompassEffect.md)/[BillboardEffect](BillboardEffect.md)) gets to rewrite `node_transform` and `node_state` first via `cull_callback()` before it's applied; `_node_reader.check_cached(false)` is called afterward since the callback may have invalidated cached node properties.
  4. Composes the (possibly-rewritten) node state into `_state`.
  5. If `clip-plane-cull` is enabled, folds any `ClipPlaneAttrib`/`OccluderEffect` on this node into `_cull_planes` via `CullPlanes::apply_state()`.
- **`apply_transform()`** composes a transform into `_net_transform`, and — only if there's an active `_view_frustum` or non-empty `_cull_planes` — also inverse-transforms those into the new node's local coordinate space (`xform()` by the node transform's inverse) so descendant-level culling tests stay in local space. If the node's transform is **singular** (non-invertible), frustum/plane culling is abandoned entirely from that point down (`_view_frustum = nullptr`, `_cull_planes = CullPlanes::make_empty()`) rather than crashing — a documented degradation, not a bug.
- **`is_in_view_impl()`** is the real frustum test: intersects `_view_frustum` against the node's bounding volume (`as_geometric_bounding_volume()`), with three outcomes: no intersection → cull (return false), unless `fake_view_frustum_cull` debug mode is on, in which case it's forced to draw in flat red wireframe instead of actually culled (non-`NDEBUG` builds only); fully contained → clears `_view_frustum` so descendants skip the frustum test entirely (a real optimization, not just debug); partial → keeps testing at deeper levels, **unless** `PandaNode::is_final()` is set on this node, which force-stops further frustum narrowing (the "final" flag is a user assertion that everything below should be considered visible from here). The same three-way logic, with the same `is_final()` short-circuit and `fake_view_frustum_cull` fallback, is repeated for `_cull_planes->do_cull()` against clip planes/occluders.
- `is_in_view()` (public) additionally checks `is_this_node_hidden(camera_mask)` is *not* what gates recursion into children by itself — a hidden node still recurses into children (a node can be individually hidden — e.g. via `NodePath::hide()` — while its children remain visible if reparented elsewhere, though in practice `is_this_node_hidden` is what `traverse_below()` checks before calling `add_for_draw()`, not before recursing).

## API

| Signature | Notes |
|---|---|
| `CullTraverserData(const NodePath &start, const TransformState *net_transform, const RenderState *state, GeometricBoundingVolume *view_frustum, Thread *current_thread)` | root constructor |
| `CullTraverserData(const CullTraverserData &parent, PandaNode *child)` | child-step constructor |
| `PandaNode *node() const` | |
| `PandaNodePipelineReader *node_reader()` | pipelined-safe access to the node's cached properties |
| `NodePath get_node_path() const` | lazily reconstructed, see behavior notes |
| `CPT(TransformState) get_modelview_transform(const CullTraverser *trav) const` | net transform composed with camera's world transform |
| `CPT(TransformState) get_internal_transform(const CullTraverser *trav) const` | modelview transform further composed into the GSG's internal coordinate system |
| `const TransformState *get_net_transform(const CullTraverser *trav) const` | |
| `bool is_in_view(const DrawMask &camera_mask)` | runs the full frustum/clip-plane test, mutating `_view_frustum`/`_cull_planes`/`_state` as side effects |
| `bool is_this_node_hidden(const DrawMask &camera_mask) const` | draw-mask bit test only, no bounds test |
| `void apply_transform_and_state(CullTraverser *trav)` | see behavior notes |
| `void apply_transform(const TransformState *node_transform)` | see behavior notes |
| `PandaNodePipelineReader _node_reader` | public field |
| `CPT(TransformState) _net_transform` | public field |
| `CPT(RenderState) _state` | public field |
| `PT(GeometricBoundingVolume) _view_frustum` | public field, local-space, `nullptr` once fully contained or culling disabled |
| `CPT(CullPlanes) _cull_planes` | public field |
| `DrawMask _draw_mask` | public field |
| `int _portal_depth` | public field, tracks portal-recursion nesting for [PortalClipper](PortalClipper.md) |

## Usage

Constructed and mutated entirely by [CullTraverser](CullTraverser.md)'s internal recursion — application code interacts with it only inside a `PandaNode::cull_callback()` or `RenderEffect::cull_callback()` override, where it's handed a `CullTraverserData&` to inspect or (rarely) adjust.

## See also

- [CullTraverser](CullTraverser.md) — owns and drives this structure
- [CullPlanes](CullPlanes.md) — the `_cull_planes` field's type
- [RenderState](RenderState.md), [TransformState](TransformState.md) — accumulated into `_state`/`_net_transform`
