# ColorScaleAttrib

**Source:** `panda/src/pgraph/colorScaleAttrib.h` (+ `.I`, `.cxx`)
**Inherits:** [RenderAttrib](RenderAttrib.md)

Applies a multiplicative RGBA scale to colors in the scene graph and on
vertices — distinct from [ColorAttrib](ColorAttrib.md), which *replaces*
the color; this *scales* whatever color would otherwise apply.

## Behavior notes
- Same `is_off()`/has-flags/quantize pattern as `AudioVolumeAttrib`: `_off`
  and `_has_scale` are independent (an off attrib can still carry a
  non-identity scale to apply below it); `quantize_scale()` rounds each
  component to the nearest 1/1024 to avoid unbounded state-cache growth
  from near-identical scales; `has_rgb_scale()`/`has_alpha_scale()` split
  the flag by channel group so a shader generator can skip work when only
  one group is scaled.
- `compose_impl`: if `other` is off, it wins outright; otherwise scales
  multiply componentwise. `invert_compose_impl`: divides componentwise,
  guarding each component's divide-by-zero by falling back to `1.0`.
- **`lower_attrib_can_override()` returns `true`** — unlike most
  `RenderAttrib`s (default `false`), a `ColorScaleAttrib` set on a lower
  node with a higher override value completely replaces the inherited one
  instead of composing with it. This is what lets a subtree set an
  override that blocks color scales from leaking in from above.
- Bam format was changed without bumping bam version (same rationale
  comment as `AudioVolumeAttrib`: no shipped files had this attrib yet).

## API

| Method | Notes |
|---|---|
| `static CPT(RenderAttrib) make_identity()` | Scale (1,1,1,1), not off; cached singleton |
| `static CPT(RenderAttrib) make(const LVecBase4 &scale)` | |
| `static CPT(RenderAttrib) make_off()` | Ignores inherited scale, own scale identity |
| `static CPT(RenderAttrib) make_default()` | Same as `make_identity()` |
| `bool is_off() const` | |
| `bool is_identity() const` | Not off, no scale |
| `bool has_scale() const` | |
| `bool has_rgb_scale() const` / `has_alpha_scale() const` | Per channel-group |
| `const LVecBase4 &get_scale() const` | |
| `CPT(RenderAttrib) set_scale(const LVecBase4 &scale) const` | Returns new interned attrib |

## Usage
```cpp
node_path.set_attrib(ColorScaleAttrib::make(LVecBase4(1, 1, 1, 0.5f)));  // half alpha
```

## See also
[RenderAttrib](RenderAttrib.md), [ColorAttrib](ColorAttrib.md),
[AudioVolumeAttrib](AudioVolumeAttrib.md) (identical off/scale pattern),
[pgraph README — the state pipeline](README.md#the-state-pipeline)
