# PointerEvent

**Source:** `panda/src/event/pointerEvent.h` / `.I` / `.cxx`
**Inherits:** none (plain value type)

Records one pointer-movement sample: position, delta since the previous
sample, and derived motion stats (length, direction, rotation). Analogous to
[ButtonEvent](ButtonEvent.md) but for motion rather than button state; batched
into a [PointerEventList](PointerEventList.md).

## Behavior notes

- **You do not fill in the derived fields yourself** — `_length`, `_direction`,
  `_rotation` are computed by [PointerEventList::add_event()](PointerEventList.md),
  not by `PointerEvent` itself (its own constructor just zero-initializes
  everything). Constructing a `PointerEvent` directly and expecting these to
  be meaningful will not work; always add through the list's `add_event()`.
- **`_direction` is in degrees, 0–360, with a flipped Y** — computed as
  `normalize_angle(rad_2_deg(atan2(-dy, dx)))` by the list, i.e. screen-space Y
  (down-positive) is negated to get a conventional math-style angle.
- **`_type`** (a `PointerType`, from `pointerData.h`) distinguishes mouse vs.
  stylus vs. touch etc., for platforms that report it; not always meaningful.
- **`write_datagram()`/`read_datagram()` are unimplemented** — both call
  `nassert_raise("This function not implemented yet.")`. Don't rely on
  `PointerEvent` Bam serialization; `PointerEventList` isn't registered with
  the Bam read factory either (unlike `ButtonEventList`).
- **Equality (`==`) compares position/motion fields only** — `_in_window`,
  `_xpos`, `_ypos`, `_dx`, `_dy`, `_sequence`, `_length`, `_direction`,
  `_rotation`; it ignores `_time`, `_id`, `_type`, and `_pressure`.

## API

| Field | Meaning |
|---|---|
| `bool _in_window` | Whether the pointer was inside the window for this sample |
| `int _id` | Device/pointer id (for multi-touch etc.) |
| `PointerType _type` | Mouse/stylus/touch/unknown |
| `double _xpos`, `_ypos` | Absolute position |
| `double _dx`, `_dy` | Delta from the previous sample in the list |
| `double _length` | Distance moved this sample (derived) |
| `double _direction` | Angle of movement, degrees, Y-flipped (derived) |
| `double _pressure` | Stylus/touch pressure, if available |
| `double _rotation` | Change in `_direction` from the previous sample (derived) |
| `int _sequence` | Monotonic sequence number assigned by the list |
| `double _time` | Timestamp |

| Method | Notes |
|---|---|
| `bool operator==/!=/<` | See "Behavior notes" for what `==` compares |
| `void output(std::ostream&) const` | `"In@x,y "` / `"Out@x,y "` |

## Usage

Almost always accessed through a [PointerEventList](PointerEventList.md)
rather than constructed directly — see that doc for `add_event()` and the
gesture-analysis helpers (`encircles()`, `total_turns()`).

## See also

[PointerEventList.md](PointerEventList.md) (owns the derived-field math) ·
[ButtonEvent.md](ButtonEvent.md) · [README.md](README.md)
