# PointerEventList

**Source:** `panda/src/event/pointerEventList.h` / `.I` / `.cxx`
**Inherits:** `ParamValueBase`

An ordered batch of recent [PointerEvent](PointerEvent.md) samples — the
pointer-motion analog of [ButtonEventList](ButtonEventList.md) — plus a few
gesture-analysis helpers (encirclement detection, total angular turning,
pattern matching) built on top of the trail. Like `ButtonEventList`, it's a
`ParamValueBase` so it can be wrapped in an [EventParameter](EventParameter.md).

## Behavior notes

- **`add_event()` computes the derived motion fields for you** — `_dx`, `_dy`,
  `_length`, `_direction`, `_rotation` are all calculated relative to the
  *previous* event in the list at insertion time (zero/undefined for the
  first event). There are three overloads: from a raw `PointerData`, from
  explicit in-window/x/y, and from explicit in-window/x/y/dx/dy (the last one
  trusts the caller's delta instead of computing it from position).
- **Backed by a `pdeque`, not a `pvector`** — `pop_front()` is O(1), which
  matters because this class is meant to be used as a bounded trailing window
  (repeatedly appended to at the back and trimmed from the front) rather than
  a one-shot batch like `ButtonEventList`.
- **`encircles(x, y)`** sums signed angular deltas from `(x,y)` to each point
  in the trail, walking *backwards* from the most recent sample, stopping
  early once the running total exceeds ±360°; returns true if the trail winds
  a full loop around the given point. Needs at least 3 events to ever return
  true.
- **`total_turns(sec)`** sums `|rotation|` over events from the last `sec`
  seconds — **note the loop `while ((pos >= 0) && (_events[pos]._time >= old)) { ...; }`
  never decrements `pos`**, so as written this is an infinite loop if any
  event qualifies (a bug in the shipped 1.10.16 source, not a documentation
  simplification — verify against your checkout before relying on this
  method).
- **`match_pattern()` is explicitly unfinished** — the header/method comment
  says "This function is not implemented yet. It is a work in progress"; it
  currently always returns `0.0` regardless of input. Do not rely on it.
- **Not registered with the Bam read factory** (unlike `ButtonEventList`) —
  no `register_with_read_factory()` call appears in `config_event.cxx` for
  this class.

## API

### Construction / basic access
| Signature | Notes |
|---|---|
| `size_t get_num_events() const` / `bool empty() const` | |
| `const PointerEvent &get_event(size_t n) const` | |
| `void clear()` / `void pop_front()` | `pop_front` for trimming a sliding window |

### Adding events
| Signature | Notes |
|---|---|
| `void add_event(const PointerData &data, int seq, double time)` | From a raw device sample |
| `void add_event(bool in_win, int xpos, int ypos, int seq, double time)` | Delta computed from previous position |
| `void add_event(bool in_win, int xpos, int ypos, double xdelta, double ydelta, int seq, double time)` | Caller-supplied delta |

### Per-event field getters
`get_in_window(n)`, `get_xpos(n)`, `get_ypos(n)`, `get_dx(n)`, `get_dy(n)`,
`get_length(n)`, `get_direction(n)`, `get_rotation(n)`, `get_sequence(n)`,
`get_time(n)` — all bounds-checked (`nassertr`), all thin wrappers over the
indexed `PointerEvent`'s fields.

### Gesture analysis
| Signature | Notes |
|---|---|
| `bool encircles(int x, int y) const` | See "Behavior notes" |
| `double total_turns(double sec) const` | **Has a known infinite-loop bug in this codebase's `_pos` decrement — see "Behavior notes"** |
| `double match_pattern(const std::string &pattern, double rot, double seglen)` | **Not implemented; always returns 0.0** |

### Output
| Signature | Notes |
|---|---|
| `void output(std::ostream&) const` / `void write(std::ostream&, int indent_level = 0) const` | |

## Usage

```cpp
PointerEventList trail;
trail.add_event(/*in_win*/true, x, y, seq++, ClockObject::get_global_clock()->get_frame_time());
// ... keep appending each frame, pop_front() old samples to bound memory ...

if (trail.encircles(target_x, target_y)) {
  // user circled the target
}
```

## See also

[PointerEvent.md](PointerEvent.md) (the per-sample value type) ·
[ButtonEventList.md](ButtonEventList.md) · [README.md](README.md)
