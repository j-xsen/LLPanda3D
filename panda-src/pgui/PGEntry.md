# PGEntry

**Source:** `panda/src/pgui/pgEntry.h` / `.I` / `.cxx`
**Inherits:** [PGItem](PGItem.md) → `PandaNode`

A single- or multi-line editable text field. Internally stores/edits text as
`std::wstring` (via a `TextAssembler`) so it supports the full Unicode
character set; 8-bit `std::string` getters/setters go through the current
`TextNode`'s encoding. Supports a blinking cursor, IME candidate-string display
(for CJK input), max-length/max-width limits, obscure ("password") mode, and a
single-line overflow/scissor mode.

## Behavior notes

- **You must call `setup()` or `setup_minimal()` before real use** — the
  constructor calls `setup_minimal(10, 1)` itself so it "doesn't crash hard if
  no one calls setup()", but that's a placeholder, not a usable default.
  `setup(width, num_lines)` additionally builds a bordered frame with
  focus/no_focus/inactive `PGFrameStyle`s and a default vertical-bar cursor
  graphic; `setup_minimal()` only sizes the text area and cursor, with no
  frame decoration.
- **States are a fixed enum:** `S_focus=0, S_no_focus, S_inactive` — driven
  automatically by `update_state()` from `get_active()`/`get_focus()`, not
  settable in arbitrary combinations.
- **`set_max_width()` means different things depending on `num_lines`.** For
  `num_lines == 1` it's a hard width cap; for `num_lines > 1` it's the
  word-wrap width per line (so effective total capacity is roughly
  `max_width * num_lines`). `0` means unlimited (subject to `max_chars`).
- **`overflow_mode`** (single-line only) lets typed text extend past the
  visible frame width instead of being rejected; the entry auto-scrolls
  (`_current_padding`) to keep the cursor visible, clipped via a `NodePath`
  scissor effect on the text geometry. This is mutually exclusive in effect
  with the normal max-width rejection/overflow-event behavior.
- **`obscure_mode`** (password fields) displays a string of `*` of the same
  length instead of the literal text; width calculations use the asterisk
  string's width, not the real text's, which affects what counts as "too long."
- **Text is flattened (`flatten_strong()`) only while the entry lacks focus** —
  while focused it's assumed to change every keystroke, so flattening is
  skipped for performance and reapplied once focus is lost.
- **`accept()` calls `set_focus(false)`**; `accept_failed()` deliberately does
  not (see the commented-out line in source) — a failed accept (Enter pressed
  while `set_accept_enabled(false)`) leaves the field focused so the user can
  keep editing.
- **Arrow/Home/End keys are gated by `set_cursor_keys_active()`** (default
  true) — set false to prevent the user from repositioning the cursor via
  keyboard (e.g. for a fixed-format field).
- **A blocked "extra trailing space" is silently eaten, not treated as
  overflow** — typing a space that would push a wrapped line over `max_width`,
  when the line already ends in a space, is dropped without firing the
  `overflow-` event (see the `keystroke()` special case for `' '`).

## API

### Setup
| Signature | Notes |
|---|---|
| `void setup(PN_stdfloat width, int num_lines)` | Full setup: frame, frame styles, default cursor geometry |
| `void setup_minimal(PN_stdfloat width, int num_lines)` | Sizes text area + cursor only, no frame decoration |

### Text (narrow-string, uses focus TextNode's encoding)
| Signature | Notes |
|---|---|
| `bool set_text(const std::string&)` | Returns false if truncated |
| `std::string get_plain_text() const` | No embedded property/formatting chars |
| `std::string get_text() const` | Includes embedded formatting chars |

### Text (wide-string, encoding-independent)
| Signature | Notes |
|---|---|
| `bool set_wtext(const std::wstring&)` | Returns false if truncated; also clamps cursor position |
| `std::wstring get_plain_wtext() const` / `get_wtext() const` | |
| `bool is_wtext() const` | True if any character is non-ASCII (i.e. you should prefer the w-string API) |

### Character/graphic introspection
| Signature | Notes |
|---|---|
| `int get_num_characters() const` | Visible characters (graphics count as 1, wordwrap newlines don't count) |
| `wchar_t get_character(int n) const` | 0 if position `n` is a graphic |
| `const TextGraphic *get_graphic(int n) const` | `nullptr` if position `n` is a character |
| `const TextProperties &get_properties(int n) const` | Properties in effect at position `n` |

### Cursor
| Signature | Notes |
|---|---|
| `void set_cursor_position(int)` / `int get_cursor_position() const` | |
| `PN_stdfloat get_cursor_X() const` / `get_cursor_Y() const` | Node-space position of the cursor graphic |
| `void set_blink_rate(PN_stdfloat)` / `get_blink_rate() const` | Blinks/sec; `0` = solid, no blink |
| `NodePath get_cursor_def()` | Attach custom cursor geometry here |
| `void clear_cursor_def()` | Removes children, gives a fresh empty node |
| `void set_cursor_keys_active(bool)` / `get_cursor_keys_active() const` | Gate arrow/Home/End |

### Limits & modes
| Signature | Notes |
|---|---|
| `void set_max_chars(int)` / `get_max_chars() const` | `0` = unlimited |
| `void set_max_width(PN_stdfloat)` / `get_max_width() const` | See "Behavior notes"; `0` = unlimited |
| `void set_num_lines(int)` / `get_num_lines() const` | |
| `void set_obscure_mode(bool)` / `get_obscure_mode() const` | Password-style masking |
| `void set_overflow_mode(bool)` / `get_overflow_mode() const` | Single-line only; see "Behavior notes" |
| `void set_accept_enabled(bool)` | Enable/disable Enter-to-accept |

### IME candidate display
| Signature | Notes |
|---|---|
| `void set_candidate_active(const std::string&)` / `get_candidate_active()` | `TextProperties` name for the actively-selected part of a candidate string |
| `void set_candidate_inactive(const std::string&)` / `get_candidate_inactive()` | `TextProperties` name for the rest of the candidate string |

### Per-state text rendering
| Signature | Notes |
|---|---|
| `void set_text_def(int state, TextNode *node)` / `TextNode *get_text_def(int state) const` | Overrides the `TextNode` (font/color/etc.) used for a given state; falls back to `PGItem::get_text_node()` |

### Overridden from PGItem
`set_active(bool)`, `set_focus(bool)` — both also call `update_state()`.

### Events (prefixes / event names)
| Signature |
|---|
| `static std::string get_accept_prefix()` → `"accept-"` |
| `static std::string get_accept_failed_prefix()` → `"acceptfailed-"` |
| `static std::string get_overflow_prefix()` → `"overflow-"` |
| `static std::string get_type_prefix()` → `"type-"` |
| `static std::string get_erase_prefix()` → `"erase-"` |
| `static std::string get_cursormove_prefix()` → `"cursormove-"` |
| `std::string get_accept_event(const ButtonHandle&) const` / `get_accept_failed_event(...)` | `prefix + button.get_name() + "-" + get_id()` |
| `std::string get_overflow_event()` / `get_type_event()` / `get_erase_event()` / `get_cursormove_event() const` | `prefix + get_id()` |

## Events

| Event | Fires when | Params |
|---|---|---|
| `accept-<button>-<id>` | Enter pressed with accept enabled | `PGMouseWatcherParameter` |
| `acceptfailed-<button>-<id>` | Enter pressed with accept disabled | `PGMouseWatcherParameter` |
| `overflow-<id>` | typed text would exceed `max_chars`/`max_width`, or arrow key hits an end with nothing to move into (i.e. also used as an "at boundary" signal) | `PGMouseWatcherParameter` |
| `type-<id>` | text extended by typing, or cursor moved via arrow/home/end successfully | `PGMouseWatcherParameter` |
| `erase-<id>` | backspace/delete removed a character | `PGMouseWatcherParameter` |
| `cursormove-<id>` | cursor position changes (incl. after any text update) | two `EventParameter`s: cursor X, cursor Y (floats) — **not** a `PGMouseWatcherParameter` |

Plus base [PGItem events](PGItem.md#events).

## Usage

```cpp
PT(PGEntry) entry = new PGEntry("username");
entry->setup(10.0f, 1);              // width in text-units, 1 line
entry->set_max_chars(32);
top_np.attach_new_node(entry);

EventHandler::get_global_event_handler()->add_hook(
  entry->get_accept_event(KeyboardButton::enter()),
  [](const Event *, void *data) {
    PGEntry *entry = (PGEntry *)data;
    // read entry->get_text()
  }, entry);
```

## See also

[PGItem.md](PGItem.md) (base frame/state/focus machinery) · [README.md](README.md)
