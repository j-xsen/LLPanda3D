# ColorAttrib

**Source:** `panda/src/pgraph/colorAttrib.h` (+ `.I`, `.cxx`)
**Inherits:** [RenderAttrib](RenderAttrib.md)

Indicates what color should be applied to renderable geometry: the
geometry's own per-vertex color, a single flat color override, or "off"
(force white).

## Behavior notes
- `Type`: `T_vertex` (use vertex color), `T_flat` (use `get_color()` for
  everything), `T_off` (force white — distinct from `T_vertex` since it
  actively overrides rather than deferring).
- `make_vertex()` and `make_off()` each cache a shared singleton
  (`_vertex`/`_off`); `make_default()` is `make_vertex()`.
- `quantize_color()` rounds the flat color to the nearest 1/1024 on
  construction and bam-load — a deliberate precision cut so that many
  near-identical flat colors collapse to the same interned attrib instead
  of growing the state cache unboundedly.

## API

| Method | Notes |
|---|---|
| `static CPT(RenderAttrib) make_vertex()` | Use geometry's own vertex color; cached |
| `static CPT(RenderAttrib) make_flat(const LColor &color)` | Quantized to 1/1024 |
| `static CPT(RenderAttrib) make_off()` | Forces white; cached |
| `static CPT(RenderAttrib) make_default()` | `make_vertex()` |
| `Type get_color_type() const` | |
| `const LColor &get_color() const` | Meaningful only for `T_flat`/`T_off` |

## Usage
```cpp
node_path.set_attrib(ColorAttrib::make_flat(LColor(1, 0, 0, 1)));
```

## See also
[RenderAttrib](RenderAttrib.md), [ColorScaleAttrib](ColorScaleAttrib.md),
[pgraph README — the state pipeline](README.md#the-state-pipeline)
