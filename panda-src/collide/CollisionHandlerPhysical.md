# CollisionHandlerPhysical

**Source:** `panda/src/collide/collisionHandlerPhysical.h` (+ `.I`, `.cxx`)
**Inherits:** [CollisionHandlerEvent](CollisionHandlerEvent.md)
**Inherited by:** [CollisionHandlerPusher](CollisionHandlerPusher.md)
(→ [CollisionHandlerFluidPusher](CollisionHandlerFluidPusher.md)),
[CollisionHandlerFloor](CollisionHandlerFloor.md), [CollisionHandlerGravity](CollisionHandlerGravity.md)

"The abstract base class for a number of CollisionHandlers that have some
physical effect on their moving bodies: they need to update the nodes'
positions based on the effects of the collision." Adds a *target* NodePath
per collider — distinct from the collider itself — that actually gets
repositioned.

## Behavior notes

- **Collider and target are separate NodePaths, added together.**
  `add_collider(collider, target)` registers a
  [CollisionNode](CollisionNode.md) path as before, but also which NodePath's
  transform should be modified when a collision requires a physical
  response — typically the same actor's top-level node, one or more levels
  above the collider itself. An overload also takes a `DriveInterface*` so
  the handler can update the drive interface's internal position state
  in sync (important if a `DriveInterface` is also independently
  repositioning the same node).
- **Still inherits the event-throwing behavior of
  [CollisionHandlerEvent](CollisionHandlerEvent.md) unchanged** — a pusher
  or floor handler can *also* fire in/again/out patterns on top of moving
  the target.
- **Subclasses implement two hooks:** `handle_entries()` (pure virtual, does
  the actual per-collider math and target repositioning, called from
  `end_group()`) and `apply_linear_force(ColliderDef&, const LVector3&)`
  (pure virtual, applies a force — meaning differs per subclass: shove
  distance for `Pusher`, downward acceleration source for `Gravity`, etc.).
- **`get_center()`/`set_center()` is optional and handler-specific** — not
  every subclass uses it (`Pusher` largely ignores it; primarily relevant
  when a handler needs a reference point to compute a push direction that
  isn't derivable from the collision geometry alone).
- **`has_contact()` reflects whether *any* collider handled this frame
  actually made contact** — checked after `traverse()`, e.g. to decide
  whether a character is airborne.

## API

### Colliders
| Signature | Notes |
|---|---|
| `void add_collider(const NodePath &collider, const NodePath &target)` | |
| `void add_collider(const NodePath &collider, const NodePath &target, DriveInterface*)` | Keeps a `DriveInterface`'s position in sync |
| `bool remove_collider(const NodePath&)` / `bool has_collider(const NodePath&) const` / `void clear_colliders()` | |

### Center & contact
| Signature | Notes |
|---|---|
| `void set_center(const NodePath&)` / `void clear_center()` / `const NodePath &get_center() const` / `bool has_center() const` | |
| `bool has_contact() const` | Whether any collider made contact this frame |

### Subclass hooks (protected)
| Signature | Notes |
|---|---|
| `virtual bool handle_entries() = 0` | Do the physical response |
| `virtual void apply_linear_force(ColliderDef&, const LVector3&) = 0` | |
| `virtual bool validate_target(const NodePath&)` | Override to reject invalid target assignments |

## See also

[CollisionHandlerEvent.md](CollisionHandlerEvent.md) · [CollisionHandlerPusher.md](CollisionHandlerPusher.md)
· [CollisionHandlerFloor.md](CollisionHandlerFloor.md) · [CollisionHandlerGravity.md](CollisionHandlerGravity.md)
· [README.md](README.md)
