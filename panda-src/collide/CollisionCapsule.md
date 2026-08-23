# CollisionCapsule

**Source:** `panda/src/collide/collisionCapsule.h` (+ `.I`, `.cxx`)
**Inherits:** [CollisionSolid](CollisionSolid.md)

"A solid consisting of a cylinder with hemispherical endcaps, also known as a
capsule or a spherocylinder." The standard "character body" shape — rounded
ends mean no sharp-edge snagging when sliding along walls or over ledges,
unlike a plain cylinder.

## Behavior notes

- **`CollisionTube` is a deprecated alias for this exact class**
  (`panda/src/collide/collisionTube.h`: `typedef CollisionCapsule
  CollisionTube;`). "This shape was previously erroneously called
  CollisionTube" — old code/docs referencing `CollisionTube` mean this
  class; prefer `CollisionCapsule` in new code.
- **Defined by two endpoints (`point_a`, `point_b`) plus a radius** — the
  capsule is the Minkowski sum of the segment `a→b` and a sphere of that
  radius. Internally caches a derived `_mat`/`_inv_mat`/`_length` from the
  two points, recomputed via `recalc_internals()` whenever an endpoint or
  radius setter is called.
- **Supports being a *from* solid against sphere, line, ray, segment,
  capsule, and parabola** — notably *not* box (no
  `test_intersection_from_box()` override), so a capsule-vs-box pair only
  resolves if the box is the *from* solid instead.
- **Friended by [CollisionBox](CollisionBox.md)** — the box's from-capsule
  test reaches into the capsule's internals directly rather than going
  through only public API.

## API

| Signature | Notes |
|---|---|
| `explicit CollisionCapsule(const LPoint3 &a, const LPoint3 &b, PN_stdfloat radius)` | |
| `explicit CollisionCapsule(PN_stdfloat ax, PN_stdfloat ay, PN_stdfloat az, PN_stdfloat bx, PN_stdfloat by, PN_stdfloat bz, PN_stdfloat radius)` | |
| `void set_point_a(const LPoint3&)` / `const LPoint3 &get_point_a() const` | |
| `void set_point_b(const LPoint3&)` / `const LPoint3 &get_point_b() const` | |
| `void set_radius(PN_stdfloat)` / `PN_stdfloat get_radius() const` | |

## Usage

```cpp
// A ~2-unit-tall standing character capsule
PT(CollisionCapsule) body = new CollisionCapsule(
    LPoint3(0, 0, 0.5), LPoint3(0, 0, 1.5), 0.4);
PT(CollisionNode) cnode = new CollisionNode("player-body");
cnode->add_solid(body);
```

## See also

[CollisionSolid.md](CollisionSolid.md) · [CollisionBox.md](CollisionBox.md)
· [README.md](README.md)
