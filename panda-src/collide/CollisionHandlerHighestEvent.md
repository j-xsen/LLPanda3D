# CollisionHandlerHighestEvent

**Source:** `panda/src/collide/collisionHandlerHighestEvent.h`
**Inherits:** [CollisionHandlerEvent](CollisionHandlerEvent.md)

Same pattern-substitution event mechanism as
[CollisionHandlerEvent](CollisionHandlerEvent.md), but each `begin_group()`/
`end_group()` cycle only keeps the single *closest* collider (tracked via an
internal `_collider_distance` / `_closest_collider`) rather than every
detected pair — useful when you only care about "what am I closest to right
now" (e.g. a single highlighted pick target) instead of every simultaneous
overlap.

## API

Inherits [CollisionHandlerEvent](CollisionHandlerEvent.md)'s full pattern API
unchanged (`add_in_pattern()`, etc.) — only `begin_group()`/`add_entry()`/
`end_group()` are overridden internally to filter down to the closest hit
before the in/again/out diff runs.

| Signature | Notes |
|---|---|
| `CollisionHandlerHighestEvent()` | |

## See also

[CollisionHandlerEvent.md](CollisionHandlerEvent.md) · [README.md](README.md)
