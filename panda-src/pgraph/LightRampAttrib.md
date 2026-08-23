# LightRampAttrib

**Source:** `panda/src/pgraph/lightRampAttrib.h` (+ `.I`, `.cxx`)
**Inherits:** RenderAttrib

A "light ramp" is a unary operator applied to each rendered pixel's
brightness after lighting — gamma correction, HDR tone mapping, and
cartoon-style quantized shading are all light ramps. `LightRampAttrib`
selects which one (if any) is active.

## Behavior notes

- `LRT_default` clamps final lighting to `[0,1]` (standard OpenGL
  behavior); `LRT_identity` disables clamping.
- `LRT_single_threshold`/`LRT_double_threshold` quantize luminance into
  discrete bands (used for cartoon/cel shading):
  `luminance = original > threshold0 ? level0 : 0.0` (single), or a
  two-band version with `threshold1`/`level1` for values above
  `threshold1`.
- `LRT_hdr0`/`hdr1`/`hdr2` are three HDR tone-mapping curves that trade off
  how much of the display's contrast range is "stolen" to represent
  brightness above 1.0 — hdr0 steals a quarter
  (`(x³+x²+x)/(x³+x²+x+1)`), hdr1 a third (`(x²+x)/(x²+x+1)`), hdr2 a half
  (`x/(x+1)`).
- `make_default()` memoizes a single shared instance in a static
  `CPT(RenderAttrib)`, same pattern as several other simple attribs.

## API

| Signature | Notes |
|---|---|
| `enum LightRampMode` | `LRT_default`, `LRT_identity`, `LRT_single_threshold`, `LRT_double_threshold`, `LRT_hdr0`, `LRT_hdr1`, `LRT_hdr2` |
| `static CPT(RenderAttrib) make_default()` | Standard OpenGL clamp-to-[0,1] |
| `static CPT(RenderAttrib) make_identity()` | No clamping |
| `static CPT(RenderAttrib) make_single_threshold(thresh0, lev0)` | One-step quantization |
| `static CPT(RenderAttrib) make_double_threshold(thresh0, lev0, thresh1, lev1)` | Two-step quantization |
| `static CPT(RenderAttrib) make_hdr0/make_hdr1/make_hdr2()` | HDR tone mapping curves |
| `LightRampMode get_mode() const` | |
| `PN_stdfloat get_level(int n) const` / `get_threshold(int n) const` | `n` is 0 or 1; out-of-range returns 0.0 |

## Usage

```cpp
node_path.set_attrib(LightRampAttrib::make_hdr1());
node_path.set_attrib(LightRampAttrib::make_single_threshold(0.5f, 1.0f)); // cartoon shading
```

## See also

[README — the state pipeline](README.md#the-state-pipeline),
[RenderAttrib](RenderAttrib.md)
