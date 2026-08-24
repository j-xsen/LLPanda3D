# CollisionBox

**Source:** `panda/src/collide/collisionBox.h` (+ `.I`, `.cxx`)
**Inherits:** [CollisionSolid](CollisionSolid.md)

A cuboid collision volume or object, defined by a center + half-extents (or
min/max corners) at construction time. Internally built from 6
[CollisionPolygon](CollisionPolygon.md)-style planes with cached 2D
projections per face, reusing the same point-in-polygon/clip machinery as
`CollisionPolygon` (see `PointDef`/`Points`/`clip_polygon()` in the header —
effectively a specialized 6-faced polygon solid, not a simple analytic-only
box test).

## Behavior notes

- **The box's orientation is fixed at construction and does not stay
  axis-aligned to world space afterward** — `xform()` transforms the box's 8
  cached vertices and 6 planes like any other solid, so a rotated
  `CollisionNode` produces a rotated (oriented) box, not an AABB that gets
  re-fit. An always-axis-aligned box requires the owning node to remain
  unrotated.
- **Supports being a *from* solid against sphere, line, ray, segment,
  capsule, and box** — same broad connectivity as
  [CollisionSphere](CollisionSphere.md).
- **Clips against active `ClipPlaneAttrib`s per face** (`apply_clip_plane()`),
  same mechanism as [CollisionPolygon](CollisionPolygon.md) — a box under a
  node with a clip plane attrib in effect gets its rendered/tested faces
  clipped accordingly.
- **`get_dimensions()` returns the full (not half) extents** — `max - min`.

## API

| Signature | Notes |
|---|---|
| `explicit CollisionBox(const LPoint3 &center, PN_stdfloat x, PN_stdfloat y, PN_stdfloat z)` | Half-extents along each axis |
| `explicit CollisionBox(const LPoint3 &min, const LPoint3 &max)` | Corner-defined |
| `const LPoint3 &get_center() const` / `const LPoint3 &get_min() const` / `const LPoint3 &get_max() const` / `LVector3 get_dimensions() const` | |
| `int get_num_points() const` / `LPoint3 get_point(int) const` | The 8 corner vertices |
| `int get_num_planes() const` / `LPlane get_plane(int) const` | The 6 face planes |

## Usage

```cpp
PT(CollisionBox) wall = new CollisionBox(LPoint3(0, 0, 1), 5.0, 0.5, 1.0);
PT(CollisionNode) cnode = new CollisionNode("wall-collider");
cnode->add_solid(wall);
render.attach_new_node(cnode);
```

## See also

[CollisionSolid.md](CollisionSolid.md) · [CollisionPolygon.md](CollisionPolygon.md)
· [README.md](README.md)
