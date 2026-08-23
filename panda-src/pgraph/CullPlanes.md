# CullPlanes

**Source:** `panda/src/pgraph/cullPlanes.h` (+ `.I`, `.cxx`)
**Inherits:** ReferenceCount

Tracks the set of clip planes ([ClipPlaneAttrib](ClipPlaneAttrib.md)-referenced `PlaneNode`s) and occluders ([OccluderNode](OccluderNode.md)s, via [OccluderEffect](OccluderEffect.md)) that are **definitely** in effect at the current node of a [CullTraverserData](CullTraverserData.md) and all its descendants — i.e., safe to cull geometry against. Immutable/copy-on-write like the [RenderState](RenderState.md) family: every mutating-looking method (`xform()`, `apply_state()`, `remove_plane()`, …) returns a new `CPT(CullPlanes)` instead of modifying in place, reusing `this` when the refcount is 1.

## Behavior notes

- **Why "definitely in effect" matters:** a clip plane or occluder that's on now but *could* be turned off by some child node's state can't be safely culled against (a descendant might legitimately need to see past it), so `CullPlanes` only accumulates planes/occluders that are part of the *net* (accumulated, unremovable) attribute state — see `apply_state()`'s `off_attrib` parameter, which excludes any plane a subtree has explicitly turned back off.
- `make_empty()` returns a shared, ref-artificially-bumped singleton empty instance (`empty->ref()` once, permanently) specifically so copy-on-write logic never mistakes it for uniquely-owned and mutates it in place.
- `xform()` and `apply_state()` both use the same copy-on-write idiom: `if (get_ref_count() == 1) reuse `this`; else copy-construct` — cheap in the common case of one live reference per traversal level.
- Occluder processing in `apply_state()` is the most involved logic in the module: for each newly-encountered occluder it (1) rejects occluders outside the view frustum, (2) rejects back-facing occluders unless `OccluderNode::is_double_sided()`, (3) optionally rejects occluders below a configured minimum screen coverage (`OccluderNode::get_min_coverage()`) by projecting through the `Lens` and measuring projected area, (4) rejects occluders that are already fully contained within an existing occluder volume (redundant), then (5) builds a `BoundingHexahedron` frustum from the occluder's near quad extruded to `2x` the lens far distance, and stores it. `show_occluder_volumes` (config var, see [module README](README.md#config-variables)) visualizes the constructed frustum.
- `do_cull()` is the actual per-node cull test: it intersects a node's bounding volume against every tracked plane/occluder, and — as a side effect — permanently drops any plane/occluder the node's bounds are found to be *entirely* in front of (respectively behind, for occluders — note the inverted sense: an occluder volume represents the region to be culled, not kept) from the returned `CullPlanes`, since descendants no longer need to test against it. It also strips the corresponding `ClipPlaneAttrib` entry from the `state` out-parameter in that case. A node found entirely behind a clip plane (or entirely within an occluder) short-circuits with `IF_no_intersection` immediately.
- Both `_planes` and `_occluders` are keyed by `NodePath` (the light/occluder's identity), not by index — `remove_plane()`/`remove_occluder()` assert the entry exists.

## API

| Signature | Notes |
|---|---|
| `bool is_empty() const` | true if no planes and no occluders |
| `static CPT(CullPlanes) make_empty()` | shared empty singleton |
| `CPT(CullPlanes) xform(const LMatrix4 &mat) const` | returns planes/occluders transformed by `mat` |
| `CPT(CullPlanes) apply_state(trav, data, net_attrib, off_attrib, node_effect) const` | folds in new clip planes from a `ClipPlaneAttrib` and occluders from an `OccluderEffect` |
| `CPT(CullPlanes) do_cull(int &result, CPT(RenderState) &state, const GeometricBoundingVolume *node_gbv) const` | the per-node test; `result` gets `BoundingVolume::IntersectionFlags`; may rewrite `state` |
| `CPT(CullPlanes) remove_plane(const NodePath &clip_plane) const` / `remove_occluder(const NodePath &occluder) const` | |
| `void write(std::ostream &out) const` | debug dump |

## Usage

`CullPlanes` is internal traversal state, threaded through [CullTraverserData](CullTraverserData.md) — application code never constructs one directly; it's driven entirely by [CullTraverser](CullTraverser.md) as it descends the graph following `ClipPlaneAttrib`/`OccluderEffect` on nodes.

## See also

- [CullTraverser](CullTraverser.md), [CullTraverserData](CullTraverserData.md) — drive `apply_state()`/`do_cull()` during traversal
- [ClipPlaneAttrib](ClipPlaneAttrib.md), [OccluderEffect](OccluderEffect.md), [OccluderNode](OccluderNode.md), [PortalClipper](PortalClipper.md) — related visibility-culling mechanisms
