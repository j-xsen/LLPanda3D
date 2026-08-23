# CollisionSphere

**Source:** `panda/src/collide/collisionSphere.h` (+ `.I`, `.cxx`)
**Inherits:** [CollisionSolid](CollisionSolid.md)
**Inherited by:** [CollisionInvSphere](CollisionInvSphere.md)

A spherical collision volume or object — the cheapest and most common solid
for both colliders and simple targets (character capsule substitute, pickup
triggers, blast radii).

## Behavior notes

- **Supports being a *from* solid against most shape types** (sphere, line,
  ray, segment, capsule, parabola, box) — one of the more fully-connected
  solids in the double-dispatch matrix, alongside
  [CollisionBox](CollisionBox.md).
- **`get_collision_origin()` returns the center** — used by
  [CollisionHandlerQueue](CollisionHandlerQueue.md)`::sort_entries()` for
  distance ordering when this is the *from* solid.

## API

| Signature | Notes |
|---|---|
| `explicit CollisionSphere(const LPoint3 &center, PN_stdfloat radius)` | |
| `explicit CollisionSphere(PN_stdfloat cx, PN_stdfloat cy, PN_stdfloat cz, PN_stdfloat radius)` | |
| `void set_center(const LPoint3&)` / `set_center(PN_stdfloat, PN_stdfloat, PN_stdfloat)` / `const LPoint3 &get_center() const` | |
| `void set_radius(PN_stdfloat)` / `PN_stdfloat get_radius() const` | |

## Usage

```cpp
PT(CollisionSphere) trigger = new CollisionSphere(0, 0, 0, 3.0);
trigger->set_tangible(false);
```

## See also

[CollisionSolid.md](CollisionSolid.md) · [CollisionInvSphere.md](CollisionInvSphere.md)
· [README.md](README.md)
