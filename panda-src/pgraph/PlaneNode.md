# PlaneNode

**Source:** `panda/src/pgraph/planeNode.h` (+ `.I`, `.cxx`)
**Inherits:** PandaNode

A node that contains a plane (`LPlane`) — most often used as a clipping
plane (via [ClipPlaneAttrib](ClipPlaneAttrib.md)), but usable anywhere a
plane needs to be defined relative to some coordinate space in the world
(e.g. reflection planes).

## Behavior notes

- **Hidden by default**: the constructor calls `set_overall_hidden(true)`
  — a `PlaneNode` doesn't render its plane. `set_cull_callback()` is also
  called in the constructor, so `cull_callback()` still fires during
  traversal *if* the node's hidden bit is overridden to make it visible
  (e.g. `show()`ing it for debugging) — in that case it draws a yellow
  wireframe grid visualization (front-facing yellow, back-facing dim
  yellow) built by `get_viz()`, which picks whichever of two largest-axis
  basis vectors best spans the plane and draws a `num_segs=10`-per-side
  grid scaled by `_viz_scale` (default `100.0`).
- `get_viz()` caches the built front/back `Geom`s (`_front_viz`/`_back_viz`)
  and picks between them each cull pass based on which side of the plane
  the lens's nodal point is on — front/back geometry differ only in
  color. The cache is invalidated (`nullptr`'d) whenever `set_plane()`,
  `set_viz_scale()`, or `xform()` changes the plane.
- `compute_internal_bounds()` returns a `BoundingPlane` — an
  infinite-half-space bounding volume, not a finite box, reflecting that
  a plane has no natural extent.
- **Priority and sorting**: mirrors [Light](Light.md)'s pattern exactly —
  `set_priority()` bumps a global `_sort_seq` (`UpdateSeq`) so every
  `ClipPlaneAttrib` in the world knows to re-sort; used when more clip
  planes are active than hardware supports (highest-priority *n* win).
  Not pipeline-cycled, for the same reason as `Light::_priority`.
- `ClipEffect` bitmask (`set_clip_effect()`) controls *what* the plane
  clips when used as a clip plane: `CE_visible` (clips rendered
  geometry), `CE_collision` (clips collision polygons). With neither bit
  set, the plane still affects culling (objects are wholly in or wholly
  out) but doesn't perform per-pixel/per-poly clipping. Default is `~0`
  (all effects).
- `xform()` transforms the stored plane by the matrix (in addition to the
  base `PandaNode::xform()`), unlike most nodes' no-op override — like
  [Fog](Fog.md), a plane's data is geometrically meaningful and must
  follow flattening.

## API

| Method | Notes |
|---|---|
| `PlaneNode(name, LPlane plane = LPlane())` | Constructor; hidden by default |
| `set_plane(LPlane)` / `get_plane()` | Invalidates viz cache on change |
| `set_viz_scale(float)` / `get_viz_scale()` | Size of the debug-wireframe visualization; default `100.0` |
| `set_priority(int)` / `get_priority()` | Relative importance among simultaneous clip planes |
| `set_clip_effect(int)` / `get_clip_effect()` | `ClipEffect` bitmask: `CE_visible` \| `CE_collision` |
| `get_sort_seq()` (static) | Global `UpdateSeq`, watched by `ClipPlaneAttrib` |

## Usage

```cpp
NodePath render("render");
PT(PlaneNode) pn = new PlaneNode("clip", LPlane(0, 0, 1, 0));
NodePath pn_np = render.attach_new_node(pn);
render.set_clip_plane(pn_np);   // applies a ClipPlaneAttrib referencing this node
```

## See also

- [ClipPlaneAttrib](ClipPlaneAttrib.md) — the RenderAttrib that activates a `PlaneNode`
- [Light](Light.md) — same priority/sort-seq pattern
- [Fog](Fog.md) — same graph-relative-effect + transformable-data pattern
