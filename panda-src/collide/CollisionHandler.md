# CollisionHandler

**Source:** `panda/src/collide/collisionHandler.h` (+ `.I`)
**Inherits:** `TypedReferenceCount`
**Inherited by:** [CollisionHandlerQueue](CollisionHandlerQueue.md),
[CollisionHandlerEvent](CollisionHandlerEvent.md)
(→ [CollisionHandlerHighestEvent](CollisionHandlerHighestEvent.md),
[CollisionHandlerPhysical](CollisionHandlerPhysical.md))

"The abstract interface to a number of classes that decide what to do when a
collision is detected." One of these must be assigned (via
[CollisionTraverser](CollisionTraverser.md)`::add_collider()`) to every
collider — it's what turns a raw geometric hit into a game effect (an event,
a queued entry, a shove).

## Behavior notes

- **Called in a strict `begin_group()` → `add_entry()`* → `end_group()`
  sequence, once per collider per `traverse()` pass.** The base class's
  `begin_group()`/`end_group()` are no-ops; subclasses override them to
  batch per-frame work (e.g. [CollisionHandlerEvent](CollisionHandlerEvent.md)
  diffs this frame's entries against last frame's in `end_group()` to derive
  in/again/out events).
- **`wants_all_potential_collidees()` (default `false`) is a traverser
  optimization hint** — most handlers only care about *confirmed* geometric
  intersections; setting this true (not exposed as a public setter on the
  base class — subclasses that need it set `_wants_all_potential_collidees`
  directly) asks the traverser to also report bounding-volume-only
  "potential" hits.
- **Not meant to be instantiated directly** (no `PUBLISHED` constructor) —
  use one of the concrete handlers.

## API

| Signature | Notes |
|---|---|
| `virtual void begin_group()` | Called before a batch of entries for one collider |
| `virtual void add_entry(CollisionEntry*)` | Called once per confirmed collision |
| `virtual bool end_group()` | Called after the batch; return value is handler-defined |

## See also

[CollisionHandlerEvent.md](CollisionHandlerEvent.md) · [CollisionHandlerQueue.md](CollisionHandlerQueue.md)
· [CollisionHandlerPhysical.md](CollisionHandlerPhysical.md) · [CollisionTraverser.md](CollisionTraverser.md)
· [README.md](README.md)
