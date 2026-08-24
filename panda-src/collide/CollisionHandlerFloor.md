# CollisionHandlerFloor

**Source:** `panda/src/collide/collisionHandlerFloor.h` (+ `.I`, `.cxx`)
**Inherits:** [CollisionHandlerPhysical](CollisionHandlerPhysical.md)

Sets the Z height of the collider to a fixed linear offset from the highest
detected collision point each frame, implementing walking on a floor of
varying height by casting a ray down from the avatar's head. No gravity or
falling simulation is performed — this is a direct snap, not physics; for
actual fall/jump velocity, see [CollisionHandlerGravity](CollisionHandlerGravity.md).

## Behavior notes

- **Picks the *highest* collision point among all this collider's entries
  each frame** (`set_highest_collision()`), then sets the target's Z to
  `highest_z + offset`. Casting the collider (typically a
  [CollisionRay](CollisionRay.md)) downward from above the avatar's actual
  feet, with `offset` roughly matching the avatar's height above its own
  origin, is the standard setup.
- **`reach` limits how far below the ray's origin a hit still counts** —
  keeps the avatar from snapping down into a distant lower floor it hasn't
  actually walked over yet (e.g. through a gap).
- **`max_velocity` caps how fast the Z position can change per frame** —
  smooths sudden steps (stairs, curbs) into a bounded climb/descent rate
  instead of an instant teleport; `0` (the implicit default before
  configuration) effectively means uncapped/instant.
- **No falling when there's no floor beneath** — unlike
  [CollisionHandlerGravity](CollisionHandlerGravity.md), if a frame's ray
  finds no hit within `reach`, the target is not moved that frame
  rather than beginning to fall.

## API

| Signature | Notes |
|---|---|
| `CollisionHandlerFloor()` | |
| `void set_offset(PN_stdfloat)` / `PN_stdfloat get_offset() const` | Z offset above the highest hit |
| `void set_reach(PN_stdfloat)` / `PN_stdfloat get_reach() const` | Max distance below the ray origin still considered |
| `void set_max_velocity(PN_stdfloat)` / `PN_stdfloat get_max_velocity() const` | Caps per-frame Z change; `0` = uncapped |

## Usage

```cpp
PT(CollisionRay) ray = new CollisionRay(0, 0, 0, 0, 0, -1);
PT(CollisionNode) cnode = new CollisionNode("floor-ray");
cnode->add_solid(ray);
NodePath ray_np = avatar_np.attach_new_node(cnode);
ray_np.set_z(4.0);  // cast from above the avatar's head

PT(CollisionHandlerFloor) floor = new CollisionHandlerFloor();
floor->set_offset(0.0);  // avatar's own origin is already at foot level
floor->add_collider(ray_np, avatar_np);
ctrav.add_collider(ray_np, floor);
```

## See also

[CollisionHandlerPhysical.md](CollisionHandlerPhysical.md) · [CollisionHandlerGravity.md](CollisionHandlerGravity.md)
· [CollisionRay.md](CollisionRay.md) · [README.md](README.md)
