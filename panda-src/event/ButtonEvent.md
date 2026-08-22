# ButtonEvent

**Source:** `panda/src/event/buttonEvent.h` / `.I` / `.cxx`
**Inherits:** none (plain value type)

Records one low-level input transition: a physical button going up/down, a
repeat, a Unicode keystroke, an IME candidate-selection update, or a mouse
move-within-region marker. This is lower-level than the [Event](Event.md)
system's named string events (e.g. `PGItem`'s `press-<button>-<id>`) — it's
the raw record the input/data-graph layer produces, batched into a
[ButtonEventList](ButtonEventList.md) and typically consumed to *generate*
those higher-level named events, or to drive a `ModifierButtons` tracker.
Per the header comment: **"This API should not be considered stable and may
change in a future version of Panda3D."**

## Behavior notes

- **A keystroke is not a button event, conceptually**, even though it shares
  this class: it doesn't necessarily correspond to a physical key (e.g. "A" =
  shift+"a" combined), has no separate up/down pair, and is defined over the
  full Unicode range rather than the fixed `ButtonHandle` registry. Check
  `get_type() == T_keystroke` and read `get_keycode()`, not `get_button()`,
  for these.
- **`T_resume_down`** exists specifically for regaining window focus: if a
  modifier key (shift, control) is physically held down when the window
  regains focus, the OS won't send a fresh "down" — this event says "treat it
  as down now" without implying a fresh press moment. Safe to ignore for
  non-modifier keys.
- **`T_raw_down`/`T_raw_up`** carry the *untransformed* scan code as if on a US
  QWERTY layout, sent alongside the normal (possibly layout-remapped)
  `T_down`/`T_up` — useful for layout-independent bindings like WASD movement.
- **Equality (`==`) deliberately ignores `_time`** — two events are equal if
  button/keycode/type match, regardless of when they occurred.
- **Only some fields are meaningful per `Type`** — `_button` is set for
  `T_down`/`T_resume_down`/`T_up`/`T_repeat`/`T_raw_down`/`T_raw_up`; `_keycode`
  for `T_keystroke`; `_candidate_string`/`_highlight_start`/`_highlight_end`/`_cursor_pos`
  for `T_candidate`. Reading the "wrong" field for a given type just returns
  whatever default/stale value is there — there's no assertion guarding this.
- **`write_datagram()`/`read_datagram()` serialize the button by *name*, not
  index** — deliberately, since a `ButtonHandle`'s numeric index isn't stable
  across sessions but its name is (looked up via `ButtonRegistry` on read).

## API

### Type
```cpp
enum Type { T_down, T_resume_down, T_up, T_repeat, T_keystroke,
            T_candidate, T_move, T_raw_down, T_raw_up };
```

### Construction
| Signature | Notes |
|---|---|
| `ButtonEvent()` | `T_down`, `ButtonHandle::none()` |
| `ButtonEvent(ButtonHandle button, Type type, double time = now)` | For down/resume_down/up/repeat/raw_down/raw_up |
| `ButtonEvent(int keycode, double time = now)` | `T_keystroke` |
| `ButtonEvent(const std::wstring &candidate_string, size_t highlight_start, highlight_end, cursor_pos)` | `T_candidate` |

### Accessors
| Signature | Notes |
|---|---|
| `ButtonHandle get_button() const` | |
| `int get_keycode() const` | |
| `Type get_type() const` | |
| `double get_time() const` | OS-reported time, matches `ClockObject::get_global_clock()`'s epoch |
| `bool update_mods(ModifierButtons &mods) const` | Calls `mods.button_down()`/`button_up()` for `T_down`/`T_up`; no-op (returns false) for other types |

### Serialization
| Signature | Notes |
|---|---|
| `void write_datagram(Datagram&) const` / `void read_datagram(DatagramIterator&)` | See "Behavior notes" on button-by-name |

## Usage

```cpp
// Typically consumed from a ButtonEventList (see ButtonEventList.md), e.g.:
for (const ButtonEvent &be : list_of_events) {
  if (be.get_type() == ButtonEvent::T_down && be.get_button() == KeyboardButton::escape()) {
    // ...
  }
}
```

## See also

[ButtonEventList.md](ButtonEventList.md) (batches of these) ·
[PointerEvent.md](PointerEvent.md) (the mouse-motion analog) · [README.md](README.md)
