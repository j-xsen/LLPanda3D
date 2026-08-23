# AuxBitplaneAttrib

**Source:** `panda/src/pgraph/auxBitplaneAttrib.h` (+ `.I`, `.cxx`)
**Inherits:** [RenderAttrib](RenderAttrib.md)

Controls what the generated shader writes into a modern framebuffer's
"aux" bitplanes (extra render targets beyond color/depth) and/or the alpha
channel of the primary color buffer. **Only has effect when shader
generation is enabled** — otherwise it's inert.

## Behavior notes
- `ABO_glow`: copies the glow map into the primary color buffer's alpha
  channel (zero if no glow map). Mutually exclusive with alpha
  blending/testing in practice — using either overrides this flag, since
  the alpha channel can't carry both glow and blend/test data at once.
- `ABO_aux_normal`: writes camera-space normal into the R,G channels of
  the first aux bitplane.
- `ABO_aux_modelz`: writes the clip-space Z of the model's center (post
  perspective-divide) into the B channel of the first aux bitplane.
- `ABO_aux_glow`: copies the glow map into the alpha channel of the first
  aux bitplane (zero if none).
- `make()` (no args) caches a single shared "no outputs" default instance
  (`_default`, separate from the interning-registry `make_default()`,
  though both end up representing outputs `0`).

## API

| Method | Notes |
|---|---|
| `static CPT(RenderAttrib) make()` | No outputs; cached singleton |
| `static CPT(RenderAttrib) make(int outputs)` | Bitwise-OR of `AuxBitplaneOutput` flags |
| `static CPT(RenderAttrib) make_default()` | Outputs `0` |
| `int get_outputs() const` | |

`AuxBitplaneOutput` enum: `ABO_glow = 1`, `ABO_aux_normal = 2`,
`ABO_aux_glow = 4`.

## Usage
```cpp
node_path.set_attrib(AuxBitplaneAttrib::make(
    AuxBitplaneAttrib::ABO_aux_normal | AuxBitplaneAttrib::ABO_aux_glow));
```

## See also
[RenderAttrib](RenderAttrib.md), [pgraph README — the state pipeline](README.md#the-state-pipeline)
