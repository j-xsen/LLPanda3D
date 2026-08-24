# CollisionHandlerPusher

**Source:** `panda/src/collide/collisionHandlerPusher.h` (+ `.I`, `.cxx`)
**Inherits:** [CollisionHandlerPhysical](CollisionHandlerPhysical.md)
**Inherited by:** [CollisionHandlerFluidPusher](CollisionHandlerFluidPusher.md)

Applies a corrective push to any object attempting to move into solid
geometry — the simplest form of physical collision response. The default
choice for allowing a player to walk around solid geometry without passing
through it.

## Behavior notes

- **Accumulates one *net* shove vector per collider per frame, summed from
  every simultaneous tangible collision, then applies it once** —
  `handle_entries()` walks each collider's `Entries` and sums
  `sd._vector * sd._length` contributions before moving the target, so
  colliding with two walls at once produces a combined push rather than two
  separate corrections.
- **`set_horizontal(true)` zeroes out the vertical (Z) component of the
  push**, keeping the target sliding along walls without being shoved
  up/down — useful for FPS/third-person-style movement where gravity/floor
  handling is done separately. Defaults from the `pushers-horizontal`
  Config.prc variable (default `false`).
- **Only tangible solids contribute** — see
  [CollisionSolid.md](CollisionSolid.md)'s `is_tangible()` note; an
  intangible "trigger" solid still fires
  [CollisionHandlerEvent](CollisionHandlerEvent.md)-style patterns (inherited)
  but never shows up in the shove sum.
- **`apply_net_shove()`/`apply_linear_force()` are `protected virtual`** —
  a subclass ([CollisionHandlerFluidPusher](CollisionHandlerFluidPusher.md))
  can override the actual application step while reusing `handle_entries()`'s
  accumulation logic.

## API

| Signature | Notes |
|---|---|
| `CollisionHandlerPusher()` | |
| `void set_horizontal(bool)` / `bool get_horizontal() const` | Strip vertical component from the push |

Everything else (`add_collider()`, `set_center()`, event patterns, etc.) is
inherited from [CollisionHandlerPhysical](CollisionHandlerPhysical.md) /
[CollisionHandlerEvent](CollisionHandlerEvent.md).

## Usage

```cpp
PT(CollisionHandlerPusher) pusher = new CollisionHandlerPusher();
pusher->set_horizontal(true);
pusher->add_collider(player_collider_np, player_np);
ctrav.add_collider(player_collider_np, pusher);
```

## See also

[CollisionHandlerPhysical.md](CollisionHandlerPhysical.md) · [CollisionHandlerFluidPusher.md](CollisionHandlerFluidPusher.md)
· [README.md](README.md)
