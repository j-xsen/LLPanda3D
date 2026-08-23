# ShadeModelAttrib

**Source:** `panda/src/pgraph/shadeModelAttrib.h` (+ `.I`, `.cxx`)
**Inherits:** RenderAttrib

Specifies flat (per-polygon) vs. smooth (per-vertex, interpolated) shading.

## Behavior notes

Simple replace-on-compose semantics (unlike `RenderModeAttrib`/`ScissorAttrib`
in this same group) — `compose_impl()` just returns `make(other->get_mode())`.

## API

| Signature | Notes |
|---|---|
| `enum Mode` | `M_flat`, `M_smooth` |
| `static CPT(RenderAttrib) make(Mode mode)` | |
| `static CPT(RenderAttrib) make_default()` | `M_smooth` |
| `Mode get_mode() const` | |

## Usage

```cpp
node_path.set_attrib(ShadeModelAttrib::make(ShadeModelAttrib::M_flat));
```

## See also

[README — the state pipeline](README.md#the-state-pipeline),
[RenderAttrib](RenderAttrib.md)
