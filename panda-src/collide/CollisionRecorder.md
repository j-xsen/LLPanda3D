# CollisionRecorder

**Source:** `panda/src/collide/collisionRecorder.h` (+ `.I`, `.cxx`)
**Inherits:** `TypedObject`
**Inherited by:** [CollisionVisualizer](CollisionVisualizer.md)

Debug-only virtual base ("used to help debug the work the collisions system
is doing") for recording every test the traverser makes each frame — both
hits and misses, not just confirmed collisions. Only compiled in when
`DO_COLLISION_RECORDING` is defined (the normal case outside optimized
release builds).

## Behavior notes

- **Installed on a [CollisionTraverser](CollisionTraverser.md) via
  `set_recorder()`, one at a time** — attaching a new recorder detaches the
  previous one from that traverser first.
- **Three virtual hooks called around and during each `traverse()` pass:**
  `begin_traversal()`, `collision_tested(entry, detected)` (once per
  candidate pair actually tested, whether or not it hit), and
  `end_traversal()`. The base class implementation just tallies
  `_num_missed`/`_num_detected`; a subclass overrides these to do something
  with the data — [CollisionVisualizer](CollisionVisualizer.md) is the only
  concrete subclass in this module.
- **No public constructor** (`protected CollisionRecorder()`) — subclass it
  to build a custom debug recorder rather than instantiating directly.

## API

| Signature | Notes |
|---|---|
| `void output(std::ostream&) const` | Prints tallied hit/miss counts |
| `virtual void begin_traversal()` | |
| `virtual void collision_tested(const CollisionEntry&, bool detected)` | Called for every pair actually tested, hit or not |
| `virtual void end_traversal()` | |

## See also

[CollisionTraverser.md](CollisionTraverser.md) · [CollisionVisualizer.md](CollisionVisualizer.md)
· [README.md](README.md)
