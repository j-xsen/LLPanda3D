# CollisionHandlerFluidPusher

**Source:** `panda/src/collide/collisionHandlerFluidPusher.h` (+ `.I`, `.cxx`)
**Inherits:** [CollisionHandlerPusher](CollisionHandlerPusher.md)

"A CollisionHandlerPusher that makes use of timing and spatial information
from fluid collisions to improve collision response." Addresses the classic
pusher weak spot: a fast-moving collider can tunnel partway through thin
geometry within a single frame before the push-out is applied.
`CollisionHandlerFluidPusher` uses the collision's `t` (parametric time, see
[CollisionEntry.md](CollisionEntry.md)) to correct the position *as of* the
moment contact actually occurred, rather than only after the fact.

## Behavior notes

- **Overrides `add_entry()` and `handle_entries()`, not the shove-application
  step** — still reuses `CollisionHandlerPusher::apply_net_shove()`, but
  changes how entries are gathered/interpreted first, giving it access to
  `CollisionSolid`-level friend access (declared a `friend class` of both
  [CollisionSolid](CollisionSolid.md) and [CollisionEntry](CollisionEntry.md)).
- **`fluid-cap-amount` (Config.prc, default `1`) caps how many extra
  timing-based subdivisions it performs per frame** — a tuning knob for the
  cost/accuracy tradeoff versus the plain
  [CollisionHandlerPusher](CollisionHandlerPusher.md).
- **Best paired with fast/thin colliders** (bullets, small fast projectiles)
  where a plain pusher would occasionally let something punch through a
  wall in one frame; for typical walk-around-the-level player movement, the
  plain pusher is usually sufficient and cheaper.

## API

| Signature | Notes |
|---|---|
| `CollisionHandlerFluidPusher()` | |

No new public setters beyond what [CollisionHandlerPusher](CollisionHandlerPusher.md)
already provides (`set_horizontal()`, `add_collider()`, etc.).

## See also

[CollisionHandlerPusher.md](CollisionHandlerPusher.md) · [CollisionEntry.md](CollisionEntry.md)
· [README.md](README.md)
