# GraphicsWindowInputDevice

**Source:** `panda/src/display/graphicsWindowInputDevice.h` (+ `.I`, `.cxx`)
**Inherits:** `InputDevice` (external base, `panda/src/device` — not documented in this module) **Inherited by:** (none)

The virtual input device representing "the keyboard and mouse pair
associated with this window." `InputDevice` (the external base class) is
Panda's generic input-device abstraction shared with real hardware devices
(gamepads, raw HID devices, etc. — see `panda/src/device`), providing the
button-event queue (`_button_events`), pointer list (`_pointers`), and
`_lock` this class builds on. `GraphicsWindowInputDevice` doesn't talk to
hardware directly — instead, a `GraphicsWindow` platform subclass calls its
`button_down()`/`button_up()`/`set_pointer_in_window()`/etc. methods as it
receives raw OS window messages, translating them into `InputDevice`'s
generic event model. Every `GraphicsWindow` normally has exactly one of
these (see `_input_devices` in [GraphicsWindow.md](GraphicsWindow.md)),
though nothing prevents a window from adding more (e.g. one per attached
joystick, historically).

## Behavior notes

- **Construction is private; only three named constructors are public**:
  `pointer_only()`, `keyboard_only()`, `pointer_and_keyboard()` — each just
  calls the private constructor with the appropriate `(pointer, keyboard)`
  booleans, which in turn call `enable_feature(Feature::pointer)` (+
  `add_pointer(PointerType::mouse, 0)`) and/or `enable_feature(Feature::keyboard)`
  on the `InputDevice` base. There's no way to construct a
  `GraphicsWindowInputDevice` with neither feature, or with more than the
  one built-in mouse pointer, through this class's own API.
- **`_buttons_held` tracks currently-down buttons purely so `focus_lost()`
  can synthesize matching "up" events** — when a window loses focus (e.g.
  alt-tab), the OS won't necessarily deliver "up" events for keys the user
  releases while focus is elsewhere, so `focus_lost()` walks `_buttons_held`
  and emits a `T_up` `ButtonEvent` for each one, then clears the set. This
  is the only place `_buttons_held` is read.
- **`button_resume_down()` vs. `button_down()`**: `button_resume_down()`
  emits a distinct `ButtonEvent::T_resume_down` transition type rather than
  a fresh `T_down` — used when a platform detects a button was already held
  down when the window gained focus/started tracking it (so downstream code
  can distinguish "this key was already down" from "this key was just
  pressed"), but still adds it to `_buttons_held` so a later release is
  tracked the same way as a normal press.
- **`raw_button_down()`/`raw_button_up()` are a separate, parallel event
  stream** (`T_raw_down`/`T_raw_up`) from the regular `button_down()`/
  `button_up()` — these represent physical key state independent of
  modifier remapping/dead-key composition, and are not added to
  `_buttons_held` (only the "logical" button methods are), so `focus_lost()`
  does not synthesize raw up-events.
- **All mutating methods take `_lock` individually** (a `LightMutex`
  inherited from `InputDevice`) rather than requiring the caller to hold it
  — every method here is safe to call from whatever thread the windowing
  system delivers its callbacks on, independent of the render/window
  thread.
- **`candidate()` exists specifically for IME (Input Method Editor)
  support** — typing composed/candidate text for CJK (Chinese/Japanese/
  Korean) input — and is forwarded as a special `ButtonEvent` constructed
  from a highlight range + cursor position rather than a button/keycode.
- **`set_pointer_in_window()`/`set_pointer_out_of_window()`** are the
  primary way a `GraphicsWindow` reports mouse presence: entering sets
  `_type = PointerType::mouse`, position, and `_in_window = true` via
  `InputDevice::update_pointer()`; leaving clears `_in_window` and
  `_pressure`, and — only if pointer-*events* (not just position) are
  enabled (`_enable_pointer_events`) — additionally appends an out-of-window
  entry to a lazily-created `PointerEventList` (see
  [../event/PointerEventList.md](../event/PointerEventList.md)) for
  precise-timing consumers.

## API

| Signature | Notes |
|---|---|
| `static PT(GraphicsWindowInputDevice) pointer_only(GraphicsWindow *host, const string &name)` | |
| `static PT(GraphicsWindowInputDevice) keyboard_only(GraphicsWindow *host, const string &name)` | |
| `static PT(GraphicsWindowInputDevice) pointer_and_keyboard(GraphicsWindow *host, const string &name)` | Most common; used by `CallbackGraphicsWindow::create_input_device()` and `SubprocessWindow`. |
| `void button_down(ButtonHandle, double time = now)` / `void button_up(ButtonHandle, double time = now)` | |
| `void button_resume_down(ButtonHandle, double time = now)` | See behavior notes. |
| `void raw_button_down(ButtonHandle, double time = now)` / `void raw_button_up(ButtonHandle, double time = now)` | Physical-key stream, independent of the logical one. |
| `void keystroke(int keycode, double time = now)` | Text-input character event (distinct from a button press). |
| `void candidate(const wstring &candidate_string, size_t highlight_start, size_t highlight_end, size_t cursor_pos)` | IME candidate text. |
| `void focus_lost(double time = now)` | Synthesizes "up" for every button in `_buttons_held`, then clears it. |
| `INLINE PointerData get_pointer() const` | Pointer 0's current data, or a default `PointerData()` if none. |
| `void set_pointer_in_window(double x, double y, double time = now)` | |
| `void set_pointer_out_of_window(double time = now)` | |
| `INLINE void update_pointer(PointerData data, double time = now)` | Thin lock-holding wrapper around `InputDevice::update_pointer()`. |
| `INLINE void pointer_moved(double x, double y, double time = now)` | Relative-motion variant, wraps `InputDevice::pointer_moved(0, x, y, time)`. |
| `INLINE void remove_pointer(int id)` | |

*(`double time = now` above stands for the actual default expression,
`ClockObject::get_global_clock()->get_frame_time()`.)*

## See also

- [GraphicsWindow.md](GraphicsWindow.md) — owns the `_input_devices` list; also see its **TouchInfo** subsection for touch-event support (handled separately from this class, via `GraphicsWindow`'s own virtual touch methods, not through `GraphicsWindowInputDevice`).
- [../event/PointerEvent.md](../event/PointerEvent.md), [../event/PointerEventList.md](../event/PointerEventList.md) — the precise-timing event stream `set_pointer_out_of_window()` can append to.
- [../framework/README.md](../framework/README.md) — `PandaFramework::get_mouse()`/`WindowFramework::get_mouse()` build a `MouseAndKeyboard` node (see [MouseAndKeyboard.md](MouseAndKeyboard.md)) on top of a window's `GraphicsWindowInputDevice`.
