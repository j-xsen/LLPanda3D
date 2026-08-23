# FogAttrib

**Source:** `panda/src/pgraph/fogAttrib.h` (+ `.I`, `.cxx`)
**Inherits:** RenderAttrib

Applies a [Fog](Fog.md) node's falloff to the geometry at and below the
node it's attached to, or disables fog if made with `make_off()`.

## Behavior notes

Like all `RenderAttrib` subclasses, instances are only ever obtained
through the `make*()` factories (constructor is private) and are
automatically interned via `return_new()` — `make(fog)` called twice with
the same `Fog*` returns the same shared object. `compare_to_impl()` and
`get_hash_impl()` compare/hash on the raw `_fog` pointer, not fog contents
— two distinct `Fog` objects with identical falloff parameters are treated
as different attribs.

## API

| Signature | Notes |
|---|---|
| `static CPT(RenderAttrib) make(Fog *fog)` | Attrib that renders with the given fog |
| `static CPT(RenderAttrib) make_off()` | Attrib that disables fog |
| `static CPT(RenderAttrib) make_default()` | Default (equivalent to off) |
| `bool is_off() const` | True if this is an off/disable attrib (`get_fog() == nullptr`) |
| `Fog *get_fog() const` | The associated Fog, or `nullptr` if off |

## Usage

```cpp
PT(Fog) fog = new Fog("scene_fog");
fog->set_linear_range(10.0f, 100.0f);
node_path.set_attrib(FogAttrib::make(fog));
// ...
node_path.set_attrib(FogAttrib::make_off());
```

## See also

[README — the state pipeline](README.md#the-state-pipeline),
[RenderAttrib](RenderAttrib.md), [Fog](Fog.md)
