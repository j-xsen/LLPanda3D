# ButtonHandle

**Source:** `panda/src/putil/buttonHandle.h` / `.I` / `.cxx`
**Inherits:** (none — standalone `final` value type)
**Related:** [ButtonRegistry](ButtonRegistry.md) (owns the registration table this class is a handle into)

A `ButtonHandle` represents a single button from any device — keyboard key,
mouse button, gamepad button, or any other named button — as a small,
cheap-to-copy opaque index into the global [ButtonRegistry](ButtonRegistry.md).
It is not constructed with data of its own; instead it wraps an `int` index
assigned by the registry at first-registration time. Application code
normally obtains one via the [KeyboardButton/MouseButton/GamepadButton](Buttons.md)
static getters rather than registering directly.

## Behavior notes

- **Default constructor deliberately does nothing.** `ButtonHandle()` leaves
  `_index` uninitialized rather than zeroing it, because many `ButtonHandle`s
  are file-scope statics (e.g. in `KeyboardButton`) whose initializers run in
  unspecified order relative to each other; zero-initializing in the default
  constructor could stomp a value another static initializer already set.
- **`ButtonHandle(const std::string &name)` implicitly registers.** Passing a
  name coerces to a handle via `ButtonRegistry::get_button()`, which
  registers a new button if the name isn't already known. This exists so a
  string literal can be implicitly converted to a `ButtonHandle`, but the
  header recommends using the static `KeyboardButton`/`MouseButton` getters
  or `ButtonRegistry::register_button()` directly instead for most purposes.
- **`matches()` is asymmetric alias comparison.** `a.matches(b)` is true if
  `a == b`, or if `b` is `a`'s alias — but *not* the reverse (if `a` is `b`'s
  alias, `b.matches(a)` is true but `a.matches(b)` is false). Aliases are
  meant to be the more general name (e.g. `shift` is an alias for `lshift`),
  so code checking "is this generically a shift key" should call
  `some_handle.matches(KeyboardButton::shift())`.
- **`has_ascii_equivalent()` is index-range-based, not a stored flag.**
  ASCII-equivalent buttons are registered with `index` set to their ASCII
  code, so any handle with `0 < index < 128` is treated as having an ASCII
  equivalent — this is a consequence of how `ButtonRegistry::register_button()`
  allocates indices, not a separate property.
- **`none()` evaluates falsy.** `operator bool()` returns `_index != 0`, and
  index 0 is reserved for `ButtonHandle::none()` (the registry reserves 128
  slots up front, including index 0). Testing `if (handle)` is the idiomatic
  "is this a real button" check.
- **`get_name()` calls back into `ButtonRegistry::ptr()`** (except for `none()`,
  which is special-cased to `"none"`) — the handle itself stores nothing but
  an index; all other data lives in the registry.

## API

| Signature | Notes |
|---|---|
| `constexpr ButtonHandle(int index)` | Wraps a pre-existing index (e.g. returned by `get_index()`); rarely used directly |
| `ButtonHandle(const std::string &name)` | Implicit; looks up/registers via the global registry |
| `bool operator ==/!=/</<=/>/>= (const ButtonHandle&) const` | Index comparison |
| `int compare_to(const ButtonHandle&) const` | `<0`/`0`/`>0` three-way compare |
| `size_t get_hash() const` | For `phash_map` |
| `std::string get_name() const` | Registry lookup |
| `bool has_ascii_equivalent() const` / `char get_ascii_equivalent() const` | See notes |
| `ButtonHandle get_alias() const` | `none()` if no alias |
| `bool matches(const ButtonHandle &other) const` | Asymmetric — see notes |
| `constexpr int get_index() const` | Raw index; intended for non-C++ scripting bridges, not general use |
| `static constexpr ButtonHandle none()` | The "no button" sentinel |
| `operator bool () const` | `false` iff `none()` |
| `void output(std::ostream&) const` | Writes `get_name()` |

## Usage

```cpp
ButtonHandle b = KeyboardButton::shift();
if (b.matches(KeyboardButton::lshift())) {
  // false: lshift is not a match for shift via matches() in this direction
}
if (KeyboardButton::lshift().matches(KeyboardButton::shift())) {
  // true: lshift's alias is shift
}
std::cout << b << "\n";  // prints "shift"
```

## See also

[ButtonRegistry.md](ButtonRegistry.md) (the table `ButtonHandle` indexes into) ·
[Buttons.md](Buttons.md) (predefined keyboard/mouse/gamepad handles) ·
[ModifierButtons.md](ModifierButtons.md) · [README.md](README.md)
