# TimedCycle

**Source:** `panda/src/putil/timedCycle.h` / `.I` / `.cxx`
**Inherits:** none
**Inherited by:** none — a small value-type helper embedded by other classes (e.g. multitexture/sprite-style "cycle through N elements every T seconds" effects elsewhere in the engine)

Cycles an index through `[0, element_count)`, advancing one step every
`cycle_time` seconds, driven by the global clock
([`ClockObject::get_global_clock()`](ClockObject.md)). Not a task — nothing
calls it automatically each frame; the owner must call `next_element()`
whenever it wants the current index, and time-based advancement happens
lazily at that call.

## Behavior notes

- **`next_element()` is pull-based, not push-based.** There's no per-frame
  update call; instead, each call to `next_element()` computes how many
  whole cycle-intervals have elapsed since `_next_switch` and jumps forward
  that many steps at once (`increment = (current_time - _next_switch) *
  _inv_cycle_time`), so it correctly catches up even if called
  infrequently (e.g. once every few seconds it still lands on the right
  element rather than needing to be polled every frame).
- **Division by zero if `next_element()` is called with `element_count ==
  0`** (`_current_child + increment) % _element_count`). The default
  constructor leaves `_element_count` at `0` — callers using the default
  constructor **must** call `set_element_count()` before the first
  `next_element()` call, or it's undefined behavior (mod-by-zero).
- **Default constructor doesn't seed `_next_switch` from the clock.** It's
  set to the sentinel `-1`, not `get_frame_time() + cycle_time` — the
  two-argument constructor does seed it properly. `set_cycle_time()` checks
  for this `-1` sentinel and lazily initializes `_next_switch` from the
  current frame time the first time it's called on a default-constructed
  object; if you never call `set_cycle_time()` after default-construction,
  the first `next_element()` call will see a huge elapsed-time delta
  (`current_time - (-1)`) and jump forward an enormous number of increments
  in one call (still mathematically correct — `% element_count` wraps it
  back into range — but wasteful/surprising the first time).
- **`set_cycle_time()` preserves phase, not just duration** — it doesn't
  reset `_next_switch` to `now + new_cycle_time`; it shifts it by the
  *difference* (`_next_switch - old_cycle_time + new_cycle_time`), so
  changing the cycle time mid-cycle doesn't reset progress toward the next
  switch.
- **`write_datagram()`/`fillin()` are Bam-style (de)serialization helpers**
  for embedding a `TimedCycle` as a field of some other `TypedWritable`
  (link [TypedWritable.md](TypedWritable.md)) — but note `TimedCycle`
  itself is *not* a `TypedWritable`; these are plain methods a containing
  class's own `write_datagram()`/`fillin()` calls through to. Also note
  `fillin()` re-seeds the clock/switch state using
  `get_real_time()` (wall-clock time), not `get_frame_time()`, unlike the
  constructor and `next_element()` — a minor asymmetry, though both usually
  track closely.
- Uses `PN_stdfloat` for the cycle time (Panda's configurable float/double
  precision typedef) but `double` for `_next_switch`/clock comparisons.

## API

| Signature | Notes |
|---|---|
| `TimedCycle()` | `element_count` starts at 0 — must call `set_element_count()` before use |
| `TimedCycle(PN_stdfloat cycle_time, int element_count)` | Fully initialized, `_next_switch` seeded from the clock |
| `void set_element_count(int)` | |
| `void set_cycle_time(PN_stdfloat)` | Phase-preserving; asserts `cycle_time > 0` |
| `int next_element()` | Advances (possibly by more than 1 step) and returns the new current index |
| `void write_datagram(Datagram&)` / `void fillin(DatagramIterator&)` | Bam-style save/load of `_cycle_time` and `_element_count` only |

## Usage

```cpp
TimedCycle cycle(0.5, 4);   // 4 elements, one every 0.5s

// each frame, or whenever you need the current frame index:
int frame = cycle.next_element();
draw_sprite_frame(frame);
```

## See also

[ClockObject.md](ClockObject.md) (the clock this reads from) ·
[README.md](README.md)
