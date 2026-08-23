# MaterialAttrib

**Source:** `panda/src/pgraph/materialAttrib.h` (+ `.I`, `.cxx`)
**Inherits:** RenderAttrib

Indicates which `Material` (surface reflectance properties: ambient,
diffuse, specular, shininess — defined in `panda/src/gobj`, undocumented)
should be applied to geometry at and below this node. Only meaningful when
lighting is enabled (see [LightAttrib](LightAttrib.md)).

## Behavior notes

- `make()` calls `material->set_attrib_lock()`, which (per `Material`'s own
  contract, not shown here) freezes the `Material` object against further
  mutation once it's referenced by an attrib — same spirit as the
  interned-immutable pattern used throughout the state pipeline.
- `compare_to_impl()`/`get_hash_impl()` compare/hash on the raw
  `Material*` pointer, not its contents — two `Material` objects with
  identical properties are still distinct attribs unless they're the same
  object.

## API

| Signature | Notes |
|---|---|
| `static CPT(RenderAttrib) make(Material *material)` | |
| `static CPT(RenderAttrib) make_off()` | Disables material (falls back to flat/vertex color lighting) |
| `static CPT(RenderAttrib) make_default()` | Equivalent to off |
| `bool is_off() const` | True if `get_material() == nullptr` |
| `Material *get_material() const` | |

## Usage

```cpp
PT(Material) mat = new Material();
mat->set_diffuse(LColor(1, 1, 1, 1));
mat->set_shininess(32.0f);
node_path.set_attrib(MaterialAttrib::make(mat));
```

## See also

[README — the state pipeline](README.md#the-state-pipeline),
[RenderAttrib](RenderAttrib.md), [LightAttrib](LightAttrib.md)
