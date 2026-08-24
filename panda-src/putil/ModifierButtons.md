# ModifierButtons

**Source:** `panda/src/putil/modifierButtons.h` / `.I` / `.cxx`
**Inherits:** (none — standalone value type)

Tracks the up/down state of a small, dynamic set of watched
[ButtonHandle](ButtonHandle.md)s — typically shift/ctrl/alt-style modifier
keys — and can produce a stable string prefix summarizing which are
currently held (e.g. for constructing composite event names like
`"control-shift-a"`). It is a value type with copy-on-write sharing of its
button list, so copying a `ModifierButtons` is cheap until one copy adds or
removes a watched button.

## Behavior notes

- **The state bitmask caps the number of watched buttons at
  `sizeof(BitmaskType) * 8`** (`BitmaskType` is `unsigned long`, so 32 or 64
  depending on platform). `add_button()` returns `false` once that limit is
  hit, silently refusing to add more.
- **The button list is copy-on-write via a `PTA(ButtonHandle)`
  (reference-counted array).** Copying a `ModifierButtons` just copies the
  pointer; `modify_button_list()` (called before any structural change —
  `add_button()`/`remove_button()`) checks `get_ref_count() > 1` and does a
  full deep copy first if the array is shared. `operator ==`/`operator <`
  compare the list *by pointer identity* first, so two objects with
  independently-built but content-identical lists are not `==`— use
  `matches()` for value-based comparison instead.
- **`&=`/`|=`/`operator &`/`operator |` take a fast path when both sides
  share the same button-list pointer** (a straight bitmask AND/OR); when the
  lists differ, they fall back to an O(n²) per-button comparison loop — the
  source comments explicitly note this is acceptable because the button
  count is expected to stay tiny ("a handful of buttons").
- **`has_button()`/`button_down()`/`button_up()`/`is_down(ButtonHandle)` all
  match via [`ButtonHandle::matches()`](ButtonHandle.md), not `==`** — so
  reporting `button_down(KeyboardButton::lshift())` will correctly flip the
  watched `shift` bit if only the generic `shift` handle (not `lshift`) was
  added to the watch list, because `lshift.matches(shift)` is true.
- **`remove_button()` is the one exception**: it matches by exact `==`, not
  `matches()` — "you cannot remove a button by removing its alias; you have
  to remove exactly the button itself" (removing also does a bit-shift
  compaction of `_state` to close the gap left in the bitmask).
- **`set_button_list()` preserves state for buttons present in both the old
  and new lists**, resetting anything else to "up" — useful for swapping in
  a differently-ordered or differently-sized watch list without losing
  currently-held state.
- **`get_prefix()` order follows watch-list insertion order, not any fixed
  canonical order** — e.g. if `control` was added before `shift`, a
  simultaneous press yields `"control-shift-"`, not `"shift-control-"`.
  Each held button contributes `<name>-` (trailing hyphen included).

## API

| Signature | Notes |
|---|---|
| `ModifierButtons()` / copy ctor / `operator =` | Copy-on-write list sharing |
| `bool operator == / != (const ModifierButtons&) const` | Pointer-identity list compare + state compare — see notes |
| `bool operator < (const ModifierButtons&) const` | List-pointer then state, for use in ordered containers |
| `ModifierButtons operator & / \| (const ModifierButtons&) const` | Non-mutating AND/OR; also `&=`/`\|=` mutating forms |
| `void set_button_list(const ModifierButtons &other)` | Replace watch list, preserving overlapping state |
| `bool matches(const ModifierButtons &other) const` | Value-based: same set of buttons reported down |
| `bool add_button(ButtonHandle)` | `false` if already watched or at capacity |
| `bool has_button(ButtonHandle) const` | Alias-aware (`matches()`) |
| `bool remove_button(ButtonHandle)` | Exact match only — see notes |
| `int get_num_buttons() const` / `ButtonHandle get_button(int) const` (+ `MAKE_SEQ` → `get_buttons()`) | Iterate the watch list |
| `bool button_down(ButtonHandle)` / `bool button_up(ButtonHandle)` | Record a transition; `false` if the button isn't watched |
| `void all_buttons_up()` | Reset state to 0 |
| `bool is_down(ButtonHandle) const` / `bool is_down(int index) const` | Alias-aware for the handle overload |
| `bool is_any_down() const` | |
| `std::string get_prefix() const` | e.g. `"control-shift-"` |
| `void output(std::ostream&) const` / `void write(std::ostream&) const` | One-line vs. multi-line dump |

## Usage

```cpp
ModifierButtons mods;
mods.add_button(KeyboardButton::shift());
mods.add_button(KeyboardButton::control());

// on a button-down input event:
mods.button_down(event_button);   // matches lshift/rshift against shift, etc.

std::string event_name = mods.get_prefix() + event_button.get_name();
// e.g. "control-shift-a"
```

## See also

[ButtonHandle.md](ButtonHandle.md) · [Buttons.md](Buttons.md) ·
[ButtonRegistry.md](ButtonRegistry.md) · [README.md](README.md)
