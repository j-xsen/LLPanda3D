# CollisionPolygon

**Source:** `panda/src/collide/collisionPolygon.h` (+ `.I`, `.cxx`)
**Inherits:** [CollisionPlane](CollisionPlane.md)

A convex, planar, finite polygon (triangle, quad, or arbitrary N-gon) — the
finite counterpart to [CollisionPlane](CollisionPlane.md). Built from an
ordered ring of coplanar points; internally re-expressed as a 2D polygon in
the plane's own coordinate space (`to_2d()`/`_to_2d_mat`) so point-in-polygon
and clipping tests run in 2D.

## Behavior notes

- **Points must be coplanar, wound consistently, and convex — the class
  will not validate or fix this for you at runtime** (the concavity-rejection
  code in the constructor is present but commented out in the `.cxx`, so a
  concave/degenerate polygon is silently accepted and will simply produce
  wrong collision results). Call `is_valid()` (at least 3 points) and
  `is_concave()` yourself during content authoring/debugging if you build
  polygons from arbitrary data; use static `verify_points(a, b, c[, d])`
  before construction to check ahead of time.
- **`is_valid()` only checks point count (`>= 3`)** — it does *not* check
  planarity or convexity, despite the name; don't rely on it to catch a bad
  polygon shape.
- **Inherits `CollisionPlane`'s solid-half-space semantics** — the polygon
  is a bounded region *of* that infinite plane; `dist_to_plane()`, `flip()`,
  `get_normal()` all still apply to its underlying plane.
- **Clips against active `ClipPlaneAttrib`s** (`apply_clip_plane()`,
  `clip_polygon()`) — a polygon under a node with a clip plane in effect is
  tested/rendered clipped to that plane, same mechanism
  [CollisionBox](CollisionBox.md) uses per-face.
- **`get_viz()` is overridden (unlike most solids) to build actual polygon
  wireframe/solid visualization geometry** reflecting the real point ring,
  not just a bounding-shape approximation.

## `CollisionGeom` — the internal Geom-triangle variant

`CollisionGeom` (`panda/src/collide/collisionGeom.h`, not documented as its
own file — see [README.md](README.md)'s exclusions) is a private
`CollisionPolygon` subclass with no public constructor. When a collider's
`into` target is a plain renderable `GeomNode` rather than a
[CollisionNode](CollisionNode.md) full of solids,
[CollisionTraverser](CollisionTraverser.md) builds one `CollisionGeom` per
triangle on the fly to run the same polygon intersection math against raw
render geometry. A `CollisionEntry` from such a hit reports `has_into() ==
false` (there was no persistent `CollisionSolid` involved) even though the
underlying test used this class — see the `%ig` pattern token in
[CollisionHandlerEvent.md](CollisionHandlerEvent.md) for how to detect this
case from event patterns.

## API

| Signature | Notes |
|---|---|
| `CollisionPolygon(const LVecBase3 &a, const LVecBase3 &b, const LVecBase3 &c)` | Triangle |
| `CollisionPolygon(const LVecBase3 &a, const LVecBase3 &b, const LVecBase3 &c, const LVecBase3 &d)` | Quad |
| `CollisionPolygon(const LPoint3 *begin, const LPoint3 *end)` | Arbitrary N-gon from a point range |
| `size_t get_num_points() const` / `LPoint3 get_point(size_t) const` | |
| `bool is_valid() const` | Point count `>= 3` only — not a full validity check |
| `bool is_concave() const` | |
| `static bool verify_points(const LPoint3 &a, const LPoint3 &b, const LPoint3 &c)` / `verify_points(a, b, c, d)` | Pre-construction sanity check |

## Usage

```cpp
PT(CollisionPolygon) floor_tri = new CollisionPolygon(
    LPoint3(0, 0, 0), LPoint3(10, 0, 0), LPoint3(10, 10, 0), LPoint3(0, 10, 0));
if (!floor_tri->is_valid()) {
  // fewer than 3 points -- bad data
}
```

## See also

[CollisionPlane.md](CollisionPlane.md) · [CollisionBox.md](CollisionBox.md)
· [CollisionTraverser.md](CollisionTraverser.md) · [README.md](README.md)
