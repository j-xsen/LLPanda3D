# CollisionHandlerEvent

**Source:** `panda/src/collide/collisionHandlerEvent.h` (+ `.I`, `.cxx`)
**Inherits:** [CollisionHandler](CollisionHandler.md)
**Inherited by:** [CollisionHandlerHighestEvent](CollisionHandlerHighestEvent.md),
[CollisionHandlerPhysical](CollisionHandlerPhysical.md)

Throws a named [event](../event/README.md) for each collision, instead of
(or in addition to, via subclassing) doing anything physical. "Its pattern
strings must first be set via a call to `add_in_pattern()` and/or
`add_out_pattern()`" — with no patterns configured, it throws nothing.

## Behavior notes

- **Three pattern lists, matched to three collision-state transitions,
  diffed once per `end_group()`:** `in_patterns` fire for a `(from, into)`
  pair that's colliding this frame but wasn't last frame; `again_patterns`
  fire for a pair colliding both this frame *and* last frame; `out_patterns`
  fire for a pair that *stopped* colliding this frame. The diff is done by
  comparing a per-frame sorted set of entries (`_current_colliding` vs.
  `_last_colliding`) — not by any state stored on the `CollisionEntry`
  itself.
- **Pattern strings are literal text with `%xx` substitution tokens**,
  expanded per-collision to build the actual event name thrown:

  | Token | Expands to |
  |---|---|
  | `%fn` | From node's name |
  | `%in` | Into node's name (empty if no `into`, e.g. a Geom-triangle hit) |
  | `%fs` | `t` if the *from* solid is tangible, else `i` |
  | `%is` | `t` if the *into* solid is tangible (or no `into`), else `i` |
  | `%ig` | `c` if there's an into solid ("collision"), else `g` (raw geometry) |
  | `%ft` | Net tag value of `key` on the *from* NodePath: `%(key)ft` |
  | `%it` | Net tag value of `key` on the *into* NodePath: `%(key)it` |
  | `%fh` | **Suppresses the whole event** unless the *from* NodePath has net tag `key`: `%(key)fh` |
  | `%fx` | **Suppresses the whole event** if the *from* NodePath has net tag `key` (inverse of `%fh`) |
  | `%ih` | **Suppresses the whole event** unless the *into* NodePath has net tag `key` |
  | `%ix` | **Suppresses the whole event** if the *into* NodePath has net tag `key` |

  An unrecognized `%xx` logs an error via the `collide` category and is
  otherwise dropped. If the fully-expanded event name is empty, nothing is
  thrown. The event's sole parameter is `EventParameter(entry)` — the
  triggering `CollisionEntry*`.
- **`clear()` forgets the "last frame" state without throwing any `out`
  events** — the next matching collision reads as a fresh `in`. `flush()` is
  the opposite: runs an empty `begin_group()`/`end_group()` pair, which
  *does* throw `out` for everything currently tracked as colliding, then
  clears.
- **`set_in_pattern()`/`set_again_pattern()`/`set_out_pattern()` replace the
  whole list with one pattern**; `add_*_pattern()` appends to a list that can
  hold several independently-fired patterns per event type.

## API

### Patterns
| Signature | Notes |
|---|---|
| `void add_in_pattern(const std::string&)` / `set_in_pattern(...)` / `clear_in_patterns()` / `int get_num_in_patterns() const` / `std::string get_in_pattern(int) const` | |
| `void add_again_pattern(...)` / `set_again_pattern(...)` / `clear_again_patterns()` / ... | Same shape, for "still colliding" |
| `void add_out_pattern(...)` / `set_out_pattern(...)` / `clear_out_patterns()` / ... | Same shape, for "stopped colliding" |

### Lifecycle
| Signature | Notes |
|---|---|
| `void clear()` | Forget tracked state, no `out` events |
| `void flush()` | Forget tracked state, *does* throw `out` events |

## Usage

```cpp
PT(CollisionHandlerEvent) handler = new CollisionHandlerEvent();
handler->add_in_pattern("%fn-into-%in");
handler->add_out_pattern("%fn-outof-%in");
ctrav.add_collider(collider_np, handler);
// listen for "player-into-wall" / "player-outof-wall" via the event system.
```

## See also

[CollisionHandler.md](CollisionHandler.md) · [CollisionHandlerHighestEvent.md](CollisionHandlerHighestEvent.md)
· [CollisionHandlerPhysical.md](CollisionHandlerPhysical.md) · [../event/README.md](../event/README.md)
· [README.md](README.md)
