# PortalNode

**Source:** `panda/src/pgraph/portalNode.h` (+ `.I`, `.cxx`)
**Inherits:** PandaNode

Holds a single rectangular portal polygon (4 vertices, counterclockwise when viewed through the portal) connecting two "cells" of the scene graph (`_cell_in`/`_cell_out`, each a `NodePath`) for portal-based visibility culling. Sets `set_cull_callback()` in its constructors so [CullTraverser](CullTraverser.md) invokes `cull_callback()` on it every time it's visited.

## Behavior notes

- **`cull_callback()` is where portal culling actually happens** (see [PortalClipper](PortalClipper.md#behavior-notes) for the full mechanism): only proceeds if the portal `is_open()`, `_cell_out` is non-empty, a [PortalClipper](PortalClipper.md) is active on the traverser (`allow-portal-cull` on), and the traversal hasn't exceeded `get_max_depth()` (`data._portal_depth <= _max_depth`, default max depth 10 — this is the mechanism that bounds infinite recursion through mirrored/looping portal chains). On success it calls `portal_viewer->prepare_portal()`, transforms the resulting reduced frustum into `_cell_out`'s coordinate space, optionally applies left/right/top/bottom clip planes (if `is_clip_plane()`), and **directly calls `trav->traverse_below(next_data)`** to recurse into the destination cell — the portal node is responsible for continuing traversal into a graph region that may not even be a normal child of this node. Regardless of whether the portal check succeeded, `cull_callback()` always returns `true` so the node's own actual children (if any) still render normally.
- `set_visible()` is toggled internally by `cull_callback()` itself (true only if the portal passed all its checks that frame) — it's a *result* of culling, not an input; `is_open()` is the actual application-controlled flag (documented as "Python sets this based on current camera zone" — i.e., higher-level game logic, not this class, decides which portals are currently traversable).
- `from_portal_mask`/`into_portal_mask` follow the same bitmask-intersection pattern as camera draw masks: a portal is only "detected" from one object to another if `(from.from_mask & into.into_mask) != 0`. `set_into_portal_mask()` marks bounds stale specifically because the net portal mask is piggybacked onto the bounding-volume computation, not because the actual geometry changes.
- `set_portal_geom(true)` widens what counts as a portal beyond `PortalSolid`s matched by the mask — it makes *every* GeomNode in the scene tested for portal intersection, an all-or-nothing setting with no per-GeomNode granularity (GeomNode has no `into_portal_mask` of its own).
- Two constructors: `PortalNode(name)` (empty, add vertices manually) and `PortalNode(name, pos, scale)` (auto-builds a default `2*scale`-sided square around `pos` in the XZ plane).

## API

| Signature | Notes |
|---|---|
| `PortalNode(const std::string &name)` | empty portal, add vertices manually |
| `PortalNode(const std::string &name, LPoint3 pos, PN_stdfloat scale=10.0)` | auto-built square portal |
| `void set_portal_mask(PortalMask)` | sets both from/into masks at once |
| `void set_from_portal_mask(PortalMask)` / `get_from_portal_mask() const` | |
| `void set_into_portal_mask(PortalMask)` / `get_into_portal_mask() const` | marks bounds stale, see behavior notes |
| `void set_portal_geom(bool)` / `get_portal_geom() const` | all GeomNodes become portal candidates |
| `void clear_vertices()` / `add_vertex(LPoint3)` | |
| `int get_num_vertices() const` / `const LPoint3 &get_vertex(int) const` / `get_vertices()` (MAKE_SEQ) | |
| `void set_cell_in(const NodePath&)` / `get_cell_in() const` | |
| `void set_cell_out(const NodePath&)` / `get_cell_out() const` | |
| `void set_clip_plane(bool)` / `is_clip_plane()` | enables left/right/top/bottom clip planes on children |
| `void set_visible(bool)` / `is_visible()` | result flag, written by `cull_callback()` |
| `void set_max_depth(int)` / `get_max_depth()` | recursion depth cap through nested portals, default 10 |
| `void set_open(bool)` / `is_open()` | application-controlled: is this portal currently traversable |
| `virtual bool cull_callback(CullTraverser*, CullTraverserData&)` | see behavior notes |
| `virtual void enable_clipping_planes()` | sets up the 4 `PlaneNode`s used when `_clip_plane` is true |

## Usage

```cpp
PortalNode *portal = new PortalNode("door", LPoint3(0, 0, 0), 2.0);
portal->set_cell_in(room_a);
portal->set_cell_out(room_b);
portal->set_open(true);
room_a.attach_new_node(portal);
```
Enable culling with `allow-portal-cull` (see [module config vars](README.md#config-variables)).

## See also

- [PortalClipper](PortalClipper.md) — implements the actual frustum-clipping math this node drives
- [CullTraverser](CullTraverser.md), [CullTraverserData](CullTraverserData.md) — traversal machinery
- [PlaneNode](PlaneNode.md) — used internally for the optional left/right/top/bottom clip planes
