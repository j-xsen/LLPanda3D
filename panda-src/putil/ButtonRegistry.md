# ButtonRegistry / ButtonMap

**Source:** `panda/src/putil/buttonRegistry.h` / `.I` / `.cxx` · `panda/src/putil/buttonMap.h` / `.I` / `.cxx`
**Inherits:** `ButtonMap : TypedReferenceCount` — `ButtonRegistry` is standalone
**Related:** [ButtonHandle](ButtonHandle.md) (the value type this registry hands out) ·
[Buttons.md](Buttons.md) (the predefined names registered into it at static-init time)

`ButtonRegistry` is the single global table backing every [ButtonHandle](ButtonHandle.md)
in the process: it assigns each unique button name an integer index, and is
the only place that ever knows the mapping from index back to name/alias.
`ButtonMap` is an unrelated, per-device value: a `TypedReferenceCount` that
maps a device's *raw* buttons to *virtual* Panda buttons (and an optional
platform-specific display label), used by input drivers to describe how a
physical keyboard/gamepad's raw button codes correspond to Panda's button
namespace.

## Behavior notes (ButtonRegistry)

- **True process-wide singleton, lazily constructed.** `ButtonRegistry::ptr()`
  constructs the one instance on first call (`init_global_pointer()`); the
  constructor itself is private, so it truly can only self-construct.
- **Slots 0–127 are pre-reserved for ASCII equivalents.** The constructor
  fills `_handle_registry` with 128 `nullptr` entries up front. `register_button()`
  places a button at its ASCII code's index directly when an `ascii_equivalent`
  is given (erroring via `util_cat->error()` if that slot is already taken);
  otherwise it appends a new index at the end of the vector. This is why
  [ButtonHandle::has_ascii_equivalent()](ButtonHandle.md) can determine ASCII-ness
  purely from index range.
- **Re-registering the same name is not an error but returns `false`** and
  silently returns the existing handle by the out-parameter — except if the
  caller's initial `button_handle` doesn't match what's already registered,
  in which case it logs a `warning()` ("Attempt to register button ... more
  than once!") before correcting the out-parameter. This tolerates multiple
  translation units racing to register the same button name at static-init
  time.
- **`get_button(name)` auto-vivifies; `find_button(name)` does not.**
  `get_button()` registers a new button on miss; `find_button()` returns
  `ButtonHandle::none()` on miss. Prefer `find_button()` when you want to
  test for existence without side effects.
- **`look_up()` treats an out-of-range index as fatal, not recoverable** —
  `util_cat->fatal()` with the message "Is memory corrupt?", since a valid
  `ButtonHandle` should never carry an index outside what this registry has
  ever handed out.
- **Runs partly at static-init time.** The file comment notes it
  deliberately uses `util_cat->info()` (arrow syntax) instead of
  `util_cat.info()` because much of this registration happens before
  `util_cat`'s own static initialization is guaranteed complete — the arrow
  form forces lazy category init on first use.

## Behavior notes (ButtonMap)

- **Purely additive: `map_button()` on an already-mapped raw button is a
  no-op.** If `_button_map` already has an entry for `raw_button`'s index,
  the call silently does nothing rather than overwriting — first mapping
  wins.
- **`get_mapped_button_label()` is not the same as the mapped button's
  Panda name.** The label is a separate, optional, possibly localized
  display string supplied by the platform/device (e.g. from OS keyboard
  layout data); `get_mapped_button(raw).get_name()` gives the Panda event
  name instead. The `.I` comments call this out explicitly.
- **String-keyed overloads round-trip through `ButtonRegistry::find_button()`**,
  so passing an unregistered name returns `none()`/empty string rather than
  registering anything (unlike `ButtonHandle`'s implicit string constructor).

## API

### ButtonRegistry
| Signature | Notes |
|---|---|
| `static ButtonRegistry *ptr()` | Global singleton accessor |
| `bool register_button(ButtonHandle &out, const std::string &name, ButtonHandle alias = none(), char ascii_equivalent = '\0')` | `true` if newly registered; always fills `out` |
| `ButtonHandle get_button(const std::string &name)` | Registers on miss |
| `ButtonHandle find_button(const std::string &name) const` (non-const in header, but no mutation on hit) | `none()` on miss |
| `ButtonHandle find_ascii_button(char ascii_equivalent) const` | `none()` on miss |
| `void write(std::ostream &out) const` | Dumps ASCII-equivalent table + all other registered buttons (with aliases) |

### ButtonMap
| Signature | Notes |
|---|---|
| `size_t get_num_buttons() const` | |
| `ButtonHandle get_raw_button(size_t i) const` / `get_mapped_button(size_t i) const` | Indexed by insertion order |
| `const std::string &get_mapped_button_label(size_t i) const` | |
| `ButtonHandle get_mapped_button(ButtonHandle raw) const` / `(const std::string &raw_name) const` | `none()` if unmapped |
| `const std::string &get_mapped_button_label(ButtonHandle raw) const` / `(const std::string &raw_name) const` | Empty string if unmapped |
| `void map_button(ButtonHandle raw, ButtonHandle mapped, const std::string &label = "")` | Protected/public-in-C++-only (not `PUBLISHED`); first-write-wins per raw button |
| `void output(std::ostream&) const` / `void write(std::ostream&, int indent_level = 0) const` | |

## Usage

```cpp
// Registering a custom button (rare — normally done via KeyboardButton etc.)
ButtonHandle my_button;
ButtonRegistry::ptr()->register_button(my_button, "my_custom_button");

// Building a device's button map (typically done by an input driver)
PT(ButtonMap) map = new ButtonMap;
map->map_button(ButtonHandle(50), KeyboardButton::ascii_key('a'), "A");
ButtonHandle mapped = map->get_mapped_button(ButtonHandle(50));
```

## See also

[ButtonHandle.md](ButtonHandle.md) · [Buttons.md](Buttons.md) ·
[ModifierButtons.md](ModifierButtons.md) · [README.md](README.md)
