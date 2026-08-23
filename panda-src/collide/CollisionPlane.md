# CollisionPlane

**Source:** `panda/src/collide/collisionPlane.h` (+ `.I`, `.cxx`)
**Inherits:** [CollisionSolid](CollisionSolid.md)
**Inherited by:** [CollisionPolygon](CollisionPolygon.md)

An infinite half-space: everything on the back side of the plane (opposite
the normal) is solid. Cheap and exact for large flat boundaries — an
infinite ground plane, an infinite wall — where a finite
[CollisionPolygon](CollisionPolygon.md)/[CollisionBox](CollisionBox.md)
would be overkill or would need careful sizing.

## Behavior notes

- **`dist_to_plane(point)` is signed** — positive in front of the plane
  (the empty-space side), negative behind (the solid side); this is the
  same convention `CollisionPolygon` builds on since it *is* a
  `CollisionPlane` restricted to a finite convex region.
- **`flip()` reverses the plane's normal (and thus which side is solid) in
  place** — cheaper than reconstructing with a negated `LPlane`.
- **Supports being a *from* solid against sphere, line, ray, segment,
  capsule, box, and parabola** — broadly connected like
  [CollisionSphere](CollisionSphere.md)/[CollisionBox](CollisionBox.md).

## API

| Signature | Notes |
|---|---|
| `CollisionPlane(const LPlane &plane)` | |
| `LVector3 get_normal() const` | |
| `PN_stdfloat dist_to_plane(const LPoint3&) const` | Signed distance |
| `void set_plane(const LPlane&)` / `const LPlane &get_plane() const` | |
| `void flip()` | Reverses which side is solid |

## Usage

```cpp
PT(CollisionPlane) ground = new CollisionPlane(LPlane(0, 0, 1, 0));  // solid below z=0
PT(CollisionNode) cnode = new CollisionNode("ground");
cnode->add_solid(ground);
render.attach_new_node(cnode);
```

## See also

[CollisionSolid.md](CollisionSolid.md) · [CollisionPolygon.md](CollisionPolygon.md)
· [README.md](README.md)
