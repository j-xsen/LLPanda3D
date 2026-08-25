# ButtonThrower

**Source:** `panda/src/tform/buttonThrower.h` / `.I` / `.cxx`
**Inherits:** `DataNode`

`ButtonThrower` converts `button_events` arriving over the [data
graph](../dgraph/README.md) into Panda `Event`s thrown via `throw_event()` —
it's the bridge between raw device input (from a `MouseAndKeyboard` or
similar) and the rest of the engine's event-handler-based systems. It can
generate both button-name-specific events (e.g. `"escape"`) and
fixed generic events (e.g. always `"keyboard-down"` with the button name as
a parameter), and can optionally filter to only a specific allow-list of
buttons.

## Behavior notes

- **Specific and generic events are independent and both optional.**
  `_specific_flag` (default `true`) gates whether `do_specific_event()`
  fires the button-name-derived event (`_prefix + button_name`, optionally
  `+ "-repeat"`/`+ "-up"`); the four/more `_button_*_event` strings
  (default empty, meaning disabled) separately gate `do_general_event()`.
  Both can fire for the same button event, or neither.
- **`_throw_buttons_active` inverts what "processing" means.** When false
  (default), *every* button is processed (events thrown) and *nothing* is
  passed further down the data graph. When true, only buttons explicitly
  added via `add_throw_button()` are processed and consumed; every other
  button is passed through unmodified to `_button_events` for a downstream
  node to see instead. `T_up` events are the one exception: they are always
  forwarded downstream when `_throw_buttons_active` is true, even for a
  button on the throw-list, "since we always pass 'up' events" (per the
  source comment) — this avoids a consumer seeing a press but never the
  matching release.
- **`add_throw_button()` keys on `(button, exact ModifierButtons state)`
  pairs, not the button alone.** The same physical button can be registered
  multiple times with different required modifier combinations (e.g.
  `shift+a` handled specially, plain `a` passed through) — `mods.matches()`
  is what's compared, so only an *exact* modifier-state match is caught by a
  given registration.
- **Modifier prefixing only happens on down-transitions, and only for
  non-modifier buttons.** `do_transmit_data()` calls `_mods.button_down(button)`
  for `T_down`/`T_repeat` events, and prepends `_mods.get_prefix()` to the
  event name only if that call returns false (meaning the button itself is
  not one of the tracked modifiers) — so pressing shift itself never
  generates `"shift-shift"`, but pressing `a` while shift is down generates
  `"shift-a"`.
- **Raw button events (`T_raw_down`/`T_raw_up`) get their own separate
  event-name family** (`"raw-" + name`, and separate
  `_raw_button_down_event`/`_raw_button_up_event` generic events), driven by
  a scancode-based button independent of keyboard layout — distinct from
  the regular (layout-aware) button of the same physical key.
- **`do_general_event()`'s parameter shape depends on the event type**: a
  down/up/repeat/raw event gets the button name as a `std::string`
  parameter; a keystroke gets a one-character `std::wstring`; a candidate
  event gets the raw wide candidate string (not the encoded
  form/highlight/cursor tuple that [MouseWatcherParameter](MouseWatcherParameter.md)
  exposes) — callers wanting the full IME candidate detail should attach to
  a [MouseWatcherRegion](MouseWatcherRegion.md)'s `candidate()` hook
  instead, not a `ButtonThrower` event.

## API

### Construction
| Signature | Notes |
|---|---|
| `explicit ButtonThrower(const std::string &name)` | Defines `button_events` in/out wire; `_specific_flag = true`, `_throw_buttons_active = false` by default |

### Generic event names (PUBLISHED, inline)
| Signature | Notes |
|---|---|
| `set/get_button_down_event`, `_button_up_event`, `_button_repeat_event` | Fixed event name for all down/up/repeat button events |
| `set/get_keystroke_event` | Fixed event, one `std::wstring` character parameter |
| `set/get_candidate_event` | Fixed event, raw candidate `std::wstring` parameter |
| `set/get_move_event`, `_raw_button_down_event`, `_raw_button_up_event` | |

### Specific-event configuration (PUBLISHED, inline)
| Signature | Notes |
|---|---|
| `set/get_prefix(const std::string &)` | Prepended to every specific event name |
| `set/get_specific_flag(bool)` | Enable/disable button-name-derived events, default true |
| `set/get_time_flag(bool)` | Prepend event time (`ClockObject`) as first parameter |
| `void add_parameter(const EventParameter &)` | Appended to every event this node throws |
| `set/get_modifier_buttons(const ModifierButtons &)` | Which modifiers get prefixed onto specific event names |

### Button allow-list
| Signature | Notes |
|---|---|
| `set/get_throw_buttons_active(bool)` | See behavior notes — inverts pass-through semantics |
| `bool add_throw_button(const ModifierButtons &, const ButtonHandle &)` | Keyed on exact mod-state match |
| `bool remove_throw_button(...)` / `bool has_throw_button(...)` (2 overloads) | |
| `void clear_throw_buttons()` | |

## Usage

```cpp
PT(ButtonThrower) bt = new ButtonThrower("kb-events");
bt->set_prefix("kb-");           // "escape" -> "kb-escape"
bt->set_button_up_event("any-key-up");
data_root.attach_new_node(bt);   // parent below a MouseAndKeyboard node
```

## See also

[DataNode](../dgraph/DataNode.md) (base wire protocol) ·
[MouseWatcherParameter](MouseWatcherParameter.md),
[MouseWatcherRegion](MouseWatcherRegion.md) (region-scoped alternative
for IME candidate/keystroke handling) ·
[display/MouseAndKeyboard](../display/MouseAndKeyboard.md) (typical
upstream source of `button_events`)
