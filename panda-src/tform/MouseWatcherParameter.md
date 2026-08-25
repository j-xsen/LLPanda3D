# MouseWatcherParameter

**Source:** `panda/src/tform/mouseWatcherParameter.h` / `.I` / `.cxx`
**Inherits:** (plain value type, not a `TypedObject`)
**Used by:** [MouseWatcherRegion](MouseWatcherRegion.md) (passed to every callback hook), [MouseWatcher](MouseWatcher.md) (constructs and fills these)

`MouseWatcherParameter` is a small, copyable bag of "what was going on when
this happened" — sent as the argument to every
[MouseWatcherRegion](MouseWatcherRegion.md) callback (`enter_region()`,
`press()`, `keystroke()`, etc.). Which fields are meaningful depends on which
callback received it; unset fields fall back to documented defaults rather
than being an error to read.

## Behavior notes

- **Every field is presence-flagged, not defaulted to a sentinel value.**
  `_flags` is a bitmask (`F_has_button`, `F_has_mouse`, `F_has_keycode`,
  `F_has_candidate`, `F_is_outside`, `F_is_keyrepeat`); each `set_*()` call
  ORs in the corresponding flag. `has_button()`/`has_mouse()`/etc. must be
  checked before trusting `get_button()`/`get_mouse()`/etc. — e.g.
  `get_mouse()` even asserts (`nassertr`) that `has_mouse()` is true.
- **`is_outside()` only has meaning for "release" and global-keyboard
  events.** [MouseWatcher](MouseWatcher.md) sets it to indicate the mouse
  had moved off the region (for a mouse-button release) or that the event is
  a global keyboard broadcast rather than one to the region directly under
  the mouse (for keyboard press/release/keystroke sent to
  `SF_other_button`-suppressing or keyboard-flagged regions); it is
  meaningless for `enter_region()`/`press()`/`move()`.
- **The candidate string is stored and returned as `std::wstring`.**
  `get_candidate_string_encoded()` exists only to convert it to a narrow
  `std::string` via `TextEncoder`, for callers that don't want to deal with
  wide strings directly; the encoding defaults to
  `TextEncoder::get_default_encoding()`.
- **Copying is a plain field-for-field copy** — the copy constructor and
  `operator=` both just copy `_button`, `_keycode`, `_mods`, `_mouse`, and
  `_flags` (curiously, `_candidate_string`/`_highlight_start`/
  `_highlight_end`/`_cursor_pos` are *not* copied by the hand-written copy
  ctor/assignment in `mouseWatcherParameter.I`, even though `_flags` may
  still claim `F_has_candidate`) — this is a real gap between what
  `has_candidate()` says and what a *copied* parameter actually contains.

## API

| Signature | Notes |
|---|---|
| `bool has_button() const` / `ButtonHandle get_button() const` | The button (mouse or keyboard) associated with this event, if any |
| `bool is_keyrepeat() const` | True if a button-down event is keyrepeat rather than an original press |
| `bool has_keycode() const` / `int get_keycode() const` | The raw keycode for a keystroke event |
| `bool has_candidate() const` / `std::string get_candidate_string_encoded([encoding]) const` | IME candidate string (see caveat above re: copies) |
| `size_t get_highlight_start/end() const`, `get_cursor_pos() const` | IME candidate highlight range and cursor position |
| `const ModifierButtons &get_modifier_buttons() const` | Modifier keys held at the time of the event |
| `bool has_mouse() const` / `const LPoint2 &get_mouse() const` | Mouse position in normalized (-1..1) coordinates |
| `bool is_outside() const` | See behavior notes above |
| `void output(std::ostream &out) const` | Human-readable one-line dump; also reachable via `operator<<` |

## Usage

```cpp
class MyRegion : public MouseWatcherRegion {
public:
  using MouseWatcherRegion::MouseWatcherRegion;

  void press(const MouseWatcherParameter &param) override {
    if (param.has_button()) {
      nout << "pressed " << param.get_button() << " at ";
      if (param.has_mouse()) {
        nout << param.get_mouse();
      }
      nout << "\n";
    }
  }
};
```

## See also

[MouseWatcherRegion](MouseWatcherRegion.md) (receives these as callback
arguments) · [MouseWatcher](MouseWatcher.md) (constructs and populates them)
