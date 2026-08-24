# CollisionSegment

**Source:** `panda/src/collide/collisionSegment.h` (+ `.I`, `.cxx`)
**Inherits:** [CollisionSolid](CollisionSolid.md)

"A finite line segment, with two specific endpoints but no thickness. It's
similar to a CollisionRay, except it does not continue to infinity." Has an
ordering from point A to point B — "if more than a single point of the
segment is intersecting a solid, the reported intersection point is
generally the closest on the segment to point A."

## Behavior notes

- **Meant as a *from* solid**, like [CollisionRay](CollisionRay.md) — used
  for bounded raycasts where hits beyond a known max distance are
  specifically not wanted (e.g. a melee attack's reach, a tether length check).
- **Same `set_from_lens()` screen-space construction helper as
  [CollisionRay](CollisionRay.md)**, but here `point_a`/`point_b` define the
  finite extent rather than an origin/infinite-direction pair.
- **No radius**, same as `CollisionRay`.

## API

| Signature | Notes |
|---|---|
| `CollisionSegment()` | |
| `explicit CollisionSegment(const LPoint3 &a, const LPoint3 &b)` | |
| `explicit CollisionSegment(PN_stdfloat ax, PN_stdfloat ay, PN_stdfloat az, PN_stdfloat bx, PN_stdfloat by, PN_stdfloat bz)` | |
| `void set_point_a(const LPoint3&)` / `const LPoint3 &get_point_a() const` | |
| `void set_point_b(const LPoint3&)` / `const LPoint3 &get_point_b() const` | |
| `bool set_from_lens(LensNode *camera, const LPoint2 &point)` | `point_a` = camera position, `point_b` = far-plane point along the screen-space ray |

## Usage

```cpp
PT(CollisionSegment) reach = new CollisionSegment(
    LPoint3(0, 0, 1), LPoint3(0, 4, 1));  // 4-unit forward reach
```

## See also

[CollisionSolid.md](CollisionSolid.md) · [CollisionRay.md](CollisionRay.md)
· [README.md](README.md)
