# CallbackObject

**Source:** `panda/src/putil/callbackObject.h` / `.I` / `.cxx` (+ `cPointerCallbackObject.h/.I/.cxx`)
**Inherits:** `TypedReferenceCount`
**Inherited by:** `CPointerCallbackObject` (below); also `PythonCallbackObject`
(Python-callable subclass, `pythonCallbackObject.h/.cxx` — Python interop
only, not documented here) and various engine-internal callback subclasses
throughout Panda (e.g. GSG/display hooks) not covered by this module.

A generic, refcounted, subclassable callback hook. Panda uses this pattern
throughout the engine at points where application code may want to intercept
or replace built-in behavior (rendering hooks, event integration, etc.):
instead of a raw function pointer, the hook point holds a `PT(CallbackObject)`
and invokes `do_callback()` with a [CallbackData](CallbackData.md) describing
that specific call site. Subclass `CallbackObject` and override `do_callback()`
to add custom C++ hook behavior; use `CPointerCallbackObject` when a plain C
function pointer + userdata is more convenient than subclassing.

## Behavior notes

- **`do_callback()` *replaces* the default behavior at that hook point — it
  does not run alongside it.** The base implementation is a no-op. To
  preserve the original behavior while adding to it, the callback must
  explicitly call `cbdata->upcall()` (see [CallbackData](CallbackData.md)),
  which each concrete `CallbackData` subclass implements to perform whatever
  the hook point would have done by default.
- **Uses `ALLOC_DELETED_CHAIN`** (a Panda memory-pool allocator macro) for
  both `CallbackObject` and `CPointerCallbackObject` — these are expected to
  be allocated/freed frequently enough to warrant a dedicated free-list
  rather than going through the general heap allocator each time.
- **`CPointerCallbackObject::do_callback()`** is a one-line trampoline:
  `(*_func)(cbdata, _data)`. It stores the function pointer and opaque
  `void *data` at construction and cannot be reconfigured afterward (no
  setters — both are constructor-only, private members).

## API

### CallbackObject
| Signature | Notes |
|---|---|
| `virtual void do_callback(CallbackData *cbdata)` | Override point; default no-op. **Not** `PUBLISHED` (protected-by-convention C++ extension point) |
| `virtual void output(std::ostream&) const` | Default prints the dynamic type name |

### CPointerCallbackObject
| Signature | Notes |
|---|---|
| `typedef void CallbackFunction(CallbackData *cbdata, void *data);` | |
| `CPointerCallbackObject(CallbackFunction *func, void *data)` | Constructor-only configuration |

## Usage

```cpp
class MyCallback : public CallbackObject {
public:
  virtual void do_callback(CallbackData *cbdata) override {
    // ... custom behavior ...
    cbdata->upcall();  // run the original behavior too, if desired
  }
};

some_hook_point->set_callback(new MyCallback);

// or, without subclassing:
PT(CPointerCallbackObject) cb = new CPointerCallbackObject(
  [](CallbackData *cbdata, void *data) {
    // ...
  }, nullptr);
```

## See also

[CallbackData.md](CallbackData.md) · [README.md](README.md)
