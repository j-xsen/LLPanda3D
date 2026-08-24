# CallbackData

**Source:** `panda/src/putil/callbackData.h` / `.I` / `.cxx`
**Inherits:** `TypedObject`
**Inherited by:** call-site-specific subclasses defined elsewhere in the
engine (e.g. GSG/display callback data types) — none live in `putil` itself.

Abstract base class for the data payload passed to
[`CallbackObject::do_callback()`](CallbackObject.md). It holds no data of its
own; each hook point in the engine defines its own `CallbackData` subclass
carrying whatever context is relevant to that call site (e.g. the GSG, the
node being rendered, etc.), and application callbacks `static_cast`/`DCAST`
the `CallbackData*` they receive to the concrete type documented at that hook
point.

## Behavior notes

- **`upcall()` is how a callback opts back into default behavior.** The base
  implementation is a no-op; each concrete subclass overrides it to perform
  whatever the engine would have done at that hook point if no callback were
  installed. A [`CallbackObject::do_callback()`](CallbackObject.md)
  implementation that wants to augment (not replace) default behavior must
  call `cbdata->upcall()` itself — nothing does this automatically.
- **Deliberately data-free at this base level** — this is a pure extension
  point; there is no generic getter/setter API here, only `output()` and
  `upcall()`. All actual payload data is defined per-subclass outside `putil`.

## API

| Signature | Notes |
|---|---|
| `virtual void upcall()` | Default no-op; subclasses override to run default hook-point behavior |
| `virtual void output(std::ostream&) const` | Default prints the dynamic type name |

## Usage

```cpp
class MyCallback : public CallbackObject {
  virtual void do_callback(CallbackData *cbdata) override {
    // cbdata is really some hook-specific subclass, e.g.:
    // MyHookCallbackData *data = DCAST(MyHookCallbackData, cbdata);
    cbdata->upcall();  // run default behavior, then/instead do custom work
  }
};
```

## See also

[CallbackObject.md](CallbackObject.md) · [README.md](README.md)
