# ColorBlendAttrib

**Source:** `panda/src/pgraph/colorBlendAttrib.h` (+ `.I`, `.cxx`)
**Inherits:** [RenderAttrib](RenderAttrib.md)

Specifies custom framebuffer color blending (`incoming * A op fbuffer * B`)
for special effects beyond plain alpha transparency — **supersedes**
[TransparencyAttrib](TransparencyAttrib.md) when its mode isn't `M_none`.
Separate mode/operand pairs can be given for RGB and alpha channels.

## Behavior notes
- Default constructor (used by `make_off()`/`make_default()`) sets mode
  `M_none`, operands `O_one`/`O_one` — blending disabled, ordinary
  transparency rules apply instead.
- `involves_constant_color()`/`involves_color_scale()` are precomputed
  flags checked across all four operands (`_a`, `_b`, `_alpha_a`,
  `_alpha_b`) at construction, so the GSG can cheaply tell whether it needs
  to feed the attrib's constant `_color` or the current
  [ColorScaleAttrib](ColorScaleAttrib.md) into the blend equation.
- **`O_color_scale`/`O_one_minus_color_scale`/`O_alpha_scale`/
  `O_one_minus_alpha_scale`** (considered for deprecation): selecting any
  of these pulls the blend constant from the current `ColorScaleAttrib`
  instead of this attrib's own `_color`, and — important side effect —
  **disables `ColorScaleAttrib`'s normal per-vertex-color scaling**, since
  the scale is applied here in the blend equation instead.
- `O_incoming1_*` operands are for dual-source blending (a second color
  output from the fragment shader).
- Bam versioning: files before minor version 42 stored one shared mode/
  operand set for both RGB and alpha (`_alpha_mode = _mode` etc. on load),
  and the dual-source operands (`O_incoming1_color` and above) were
  encoded shifted by 4 (adjusted on read for old files).
- The one-`Mode`-arg `make()` overload is deprecated in favor of the 3- or
  6-arg forms.

## API

| Method | Notes |
|---|---|
| `static CPT(RenderAttrib) make_off()` | Mode `M_none`; blending disabled |
| `static CPT(RenderAttrib) make(Mode rgb_mode, Operand a, Operand b, const LColor &color = zero)` | Same mode/operands for RGB and alpha |
| `static CPT(RenderAttrib) make(Mode rgb_mode, Operand a, Operand b, Mode alpha_mode, Operand alpha_a, Operand alpha_b, const LColor &color = zero)` | Separate RGB/alpha blend equations |
| `static CPT(RenderAttrib) make_default()` | `M_none` |
| `Mode get_mode() const` / `get_alpha_mode() const` | |
| `Operand get_operand_a/b() const` / `get_alpha_operand_a/b() const` | |
| `LColor get_color() const` | Constant blend color, when an operand references it |
| `bool involves_constant_color() const` / `involves_color_scale() const` | Precomputed operand-derived flags |

`Mode`: `M_none`, `M_add`, `M_subtract`, `M_inv_subtract`, `M_min`,
`M_max`. `Operand`: `O_zero`, `O_one`, `O_{incoming,fbuffer}_{color,alpha}`,
`O_one_minus_*` variants, `O_constant_{color,alpha}` (+ `one_minus`),
`O_incoming_color_saturate` (operand A only), `O_incoming1_*` (dual-source),
`O_{color,alpha}_scale` (+ `one_minus`, deprecation-considered).

## Usage
```cpp
// Additive blending: incoming color + fbuffer color
node_path.set_attrib(ColorBlendAttrib::make(
    ColorBlendAttrib::M_add,
    ColorBlendAttrib::O_one, ColorBlendAttrib::O_one));
```

## See also
[RenderAttrib](RenderAttrib.md), [TransparencyAttrib](TransparencyAttrib.md),
[ColorScaleAttrib](ColorScaleAttrib.md), [pgraph README — the state pipeline](README.md#the-state-pipeline)
