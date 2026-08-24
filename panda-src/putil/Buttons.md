# KeyboardButton / MouseButton / GamepadButton / MouseData

**Source:** `panda/src/putil/keyboardButton.h/.cxx` · `mouseButton.h/.cxx` ·
`gamepadButton.h/.cxx` · `mouseData.h`

Three of these classes are not real classes — they're pure-static namespaces
grouping factory functions that return predefined [ButtonHandle](ButtonHandle.md)s
for standard input devices. None of them are constructible or hold instance
state (`MouseButton` holds static handle storage only). Every handle they
return has already been [registered](ButtonRegistry.md) by an `init_*_buttons()`
call made once from `config_putil.cxx` at library-init time; application code
just calls the static getter.

`MouseData` (`mouseData.h`) is unrelated to the button namespaces — it's a
one-line deprecated `typedef` alias for `PointerData` (see the pointer/mouse
tracking classes elsewhere in this module), kept only for source compatibility.

## Behavior notes

- **`KeyboardButton::ascii_key(char)`** doesn't return a static — it looks up
  `ButtonRegistry::ptr()->find_ascii_button()` live, since ASCII-mapped keys
  are registered at their ASCII code's registry index (see
  [ButtonRegistry](ButtonRegistry.md)).
- **A handful of keys register an ASCII equivalent alongside their name**:
  `space` (`' '`), `backspace` (`\x08`), `tab` (`\x09`), `enter` (`\x0d`),
  `escape` (`\x1b`), `del` (`\x7f`, registered under the name `"delete"`
  since `delete` is a C++ keyword). All other keyboard buttons (F-keys,
  arrows, locks, modifiers) register with no ASCII code.
- **Arrow keys are named `arrow_left`/`arrow_right`/`arrow_up`/`arrow_down`**,
  not `left`/`up`/etc. — the source comment notes `up` specifically conflicts
  with the key-*release* event name `up` that Panda's input system throws
  (a button named `up`'s press event would collide with the release-event
  convention).
- **Left/right modifier variants are registered as aliases of the generic
  name**: `lshift`/`rshift` both alias `shift`, `lcontrol`/`rcontrol` alias
  `control`, `lalt`/`ralt` alias `alt`, `lmeta`/`rmeta` alias `meta`. Use
  [`ButtonHandle::matches()`](ButtonHandle.md) to test generically (e.g. "was
  any shift key pressed") against the base name.
- **`MouseButton`'s numbered buttons are 0-indexed internally but 1-indexed
  in their names**: `_buttons[0]` registers under `"mouse1"` and is returned
  by `MouseButton::one()`; `MouseButton::button(n)` takes a 0-based index and
  returns `none()` if out of `[0, num_mouse_buttons)` range (`num_mouse_buttons == 5`).
- **`MouseButton::is_mouse_button()`** is a linear scan (5 numbered buttons +
  4 wheel directions) — the only way to classify a handle as mouse-vs-other
  from this API, since `ButtonHandle` itself carries no device-category tag.
- **`GamepadButton` covers two distinct device idioms in one namespace**:
  modern gamepad face/stick/trigger buttons (`face_a()`...`face_z()`,
  `lstick()`, `ltrigger()`, ...) *and* older flight-stick conventions
  (`trigger()`, `joystick(int)`, `hat_up/down/left/right()`) — check which
  your target device driver actually emits.

## API

| Namespace | What it provides |
|---|---|
| `KeyboardButton` | `ascii_key(char)`; named getters for whitespace/control keys (`space`, `backspace`, `tab`, `enter`, `escape`), F-keys `f1()`–`f16()`, navigation (`left/right/up/down`, `page_up/down`, `home/end/insert/del/help/menu`), and modifiers both generic and left/right-specific (`shift`, `lshift`, `rshift`, `control`, `lcontrol`, `rcontrol`, `alt`, `lalt`, `ralt`, `meta`, `lmeta`, `rmeta`, plus lock keys `caps_lock/shift_lock/num_lock/scroll_lock`, `print_screen`, `pause`). Full list: `keyboardButton.h`. |
| `MouseButton` | `button(int button_number)` (0-based, `none()` if out of range), `one()`–`five()`, `wheel_up/down/left/right()`, `is_mouse_button(ButtonHandle)` |
| `GamepadButton` | Sticks/shoulders/triggers/grips (`lstick`, `rstick`, `lshoulder`, `rshoulder`, `ltrigger`, `rtrigger`, `lgrip`, `rgrip`), d-pad (`dpad_left/right/up/down`), menu buttons (`back`, `guide`, `start`, `next`, `previous`), face buttons (`face_a`...`face_z`, `face_1`, `face_2`), and flight-stick buttons (`trigger()`, `joystick(int button_number)` zero-based, `hat_up/down/left/right()`). Full list: `gamepadButton.h`. |
| `MouseData` | `typedef PointerData MouseData;` — deprecated alias only |

## Usage

```cpp
ButtonHandle a_key = KeyboardButton::ascii_key('a');
ButtonHandle any_shift = KeyboardButton::shift();

// generic check regardless of which physical shift was pressed
bool shift_down = current_button.matches(any_shift);

if (MouseButton::is_mouse_button(current_button)) {
  // ...
}
```

## See also

[ButtonHandle.md](ButtonHandle.md) · [ButtonRegistry.md](ButtonRegistry.md) ·
[ModifierButtons.md](ModifierButtons.md) · [README.md](README.md)
