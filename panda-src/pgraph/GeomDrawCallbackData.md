# GeomDrawCallbackData

**Source:** `panda/src/pgraph/geomDrawCallbackData.h` (+ `.I`, `.cxx`)
**Inherits:** CallbackData

The `CallbackData` subclass passed to a user `CallbackObject` when [CullableObject::draw()](CullableObject.md) is drawing an object that has a draw callback set instead of drawing its `Geom` directly. Lets application/plugin code hook into the low-level draw path for one Geom (e.g. to issue raw graphics-API calls).

## Behavior notes

- Constructed fresh on the stack by `CullableObject::draw()`/`draw_callback()` for each callback invocation — not a long-lived object.
- `_lost_state` defaults to `true`: by default Panda assumes the callback leaves the GSG's state/transform in an unknown condition and will reissue whatever's needed afterward. A callback that's careful to leave GSG state exactly as found can call `set_lost_state(false)` to skip that restoration cost.
- `upcall()` — calling this from inside the callback resumes "normal" rendering of the wrapped object's `_geom` (as if no callback were set). It's a no-op if the object has no `_geom` (relevant when the callback is invoked on a `CallbackNode`, which has no Geom at all).

## API

| Signature | Notes |
|---|---|
| `GeomDrawCallbackData(CullableObject *obj, GraphicsStateGuardianBase *gsg, bool force)` | constructed internally by CullableObject |
| `CullableObject *get_object() const` | |
| `GraphicsStateGuardianBase *get_gsg() const` | |
| `bool get_force() const` | whether non-resident data should be forced in |
| `void set_lost_state(bool)` / `bool get_lost_state() const` | default true; see behavior notes |
| `virtual void upcall()` | resumes default drawing of `_obj->_geom` |
| `virtual void output(std::ostream &out) const` | |

## Usage

```cpp
class MyCallback : public CallbackObject {
public:
  virtual void do_callback(CallbackData *cbdata) {
    GeomDrawCallbackData *gcbdata = DCAST(GeomDrawCallbackData, cbdata);
    // ... issue custom draw calls using gcbdata->get_gsg() ...
    gcbdata->set_lost_state(false); // if state was left untouched
  }
};
```

## See also

- [CullableObject](CullableObject.md) — constructs and drives this
