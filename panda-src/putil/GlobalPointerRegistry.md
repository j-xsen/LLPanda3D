# GlobalPointerRegistry / PointerData

**Source:** `panda/src/putil/globalPointerRegistry.h/.I/.cxx`,
`panda/src/putil/pointerData.h/.I/.cxx`
**Inherits:** none (both)
**Inherited by:** none (both)

These two classes share nothing conceptually — they're documented together
only because both files have "pointer" in the name. `GlobalPointerRegistry`
is a `TypeHandle → void*` registry (a workaround for cross-shared-library
static-data duplication). `PointerData` is a plain value struct describing
one 2D input-pointer sample (mouse/touch/stylus position). Don't confuse
"pointer" in the C++ sense (the registry) with "pointer" in the input-device
sense (`PointerData`).

## GlobalPointerRegistry

A global singleton mapping `TypeHandle → void*`, used as a substitute for a
template class's `static` data member.

**Why this exists (from the header comment):** a template class's static
data member (e.g. `Foo<int>::_a`) is supposed to be one shared instance
across the whole program. But if multiple shared libraries each
instantiate `Foo<int>`, the dynamic loader is responsible for collapsing
the duplicate static-data instances into one — and on some platforms
(the header specifically calls out Linux + gcc) this sometimes fails,
leaving two independent `Foo<int>::_a` instances where some code sees one
and some code sees the other. Since `TypeHandle` values are already
globally unique (keyed by the string name passed to `init_type()`, resolved
through the single central type-registry), routing "per-type static data"
through this registry instead of an actual `static` member sidesteps the
loader bug entirely.

### Behavior notes

- **One `store_pointer()` per `TypeHandle`, ever (without an intervening
  `clear_pointer()`).** Calling `store_pointer()` twice for the same type
  without clearing first logs an `util_cat` error (not an exception/assert)
  and leaves the *original* pointer in place — the second store is
  rejected, not silently overwritten.
- **`store_pointer(type, nullptr)` is also an error** — logs and redirects
  to `clear_pointer(type)` instead of storing `nullptr`.
- **Errors are logged, not thrown/asserted** — misuse (storing twice,
  storing null, looking up `TypeHandle::none()`) degrades to an
  `util_cat->error()` log line and best-effort behavior, so bugs here can
  go unnoticed unless someone's watching the log.
- **Lazy global singleton via a raw pointer**, not a function-local
  `static` — `get_global_ptr()` explicitly avoids
  static-initialization-order issues since `GlobalPointerRegistry` itself
  may be queried very early (from other static initializers, given this
  exists to solve a static-init-adjacent problem in the first place).

### API

| Signature | Notes |
|---|---|
| `static void *get_pointer(TypeHandle)` | `nullptr` if never stored |
| `static void store_pointer(TypeHandle, void *ptr)` | Errors (logged) if called twice for the same type, or with `ptr == nullptr` |
| `static void clear_pointer(TypeHandle)` | Safe to call even if nothing was stored |

### Usage

```cpp
template<class T>
class Foo {
  static T *get_shared_data() {
    void *p = GlobalPointerRegistry::get_pointer(get_class_type());
    if (p == nullptr) {
      p = new T();
      GlobalPointerRegistry::store_pointer(get_class_type(), p);
    }
    return (T *)p;
  }
};
```

## PointerData

A plain data struct describing one sample from a 2D pointer input device
(mouse, touchscreen finger, stylus/eraser) — position, whether it's
currently inside the window, pressure, a per-pointer id (for tracking
individual fingers across a multitouch stream), and a `PointerType` enum
(`unknown`, `mouse`, `finger`, `stylus`, `eraser`). All fields are public
and directly settable (`_xpos`, `_ypos`, `_in_window`, `_pressure`, `_type`,
`_id`); the getters are a thin `PUBLISHED`/const-access convenience layer.

### Behavior notes

- **`get_in_window() == false` means position is stale/meaningless** — the
  header explicitly warns `get_x()`/`get_y()` "may not be meaningful" when
  the pointer isn't currently in the window.
- **`get_id()` is opaque** — "should not be assumed to have a specific
  meaning other than that there will not be two different pointers active
  simultaneously with the same identifier" — its only contract is
  uniqueness among concurrently-active pointers, useful for correlating a
  sequence of samples to the same physical finger/stylus across frames.
- **`get_pressure()` for a plain mouse is binary**: `1.0` if any button is
  currently down, `0.0` otherwise — not a continuous value unless the
  device is an actual pressure-sensitive one (stylus/finger).

### API

| Signature | Notes |
|---|---|
| `double get_x() const` / `get_y() const` | Meaningless if `!get_in_window()` |
| `bool get_in_window() const` | |
| `int get_id() const` | Unique among concurrently-active pointers only |
| `PointerType get_type() const` | `unknown`/`mouse`/`finger`/`stylus`/`eraser` |
| `double get_pressure() const` | `0.0`/`1.0` for a mouse; continuous for pressure-sensitive devices |
| `void output(std::ostream&) const` | |

## See also

[ModifierButtons.md](ModifierButtons.md) (`pointerData.h` includes it, but
`PointerData` doesn't otherwise use it) · [README.md](README.md)
