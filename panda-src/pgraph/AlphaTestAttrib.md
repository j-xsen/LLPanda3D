# AlphaTestAttrib

**Source:** `panda/src/pgraph/alphaTestAttrib.h` (+ `.I`, `.cxx`)
**Inherits:** [RenderAttrib](RenderAttrib.md)

Enables or disables writing a pixel to the framebuffer based on comparing
its alpha value against a reference value — the classic "alpha test" used
for cheap cutout transparency (leaves, fences, etc.) without needing
blending.

## Behavior notes
- `make_default()` returns mode `M_always` (test always passes — alpha
  test effectively off), reference alpha `1.0`.
- `reference_alpha` must be in `[0.0, 1.0]` (asserted in `make()`).
- Uses the shared `PandaCompareFunc` enum (`M_never`, `M_less`, `M_equal`,
  `M_less_equal`, `M_greater`, `M_not_equal`, `M_greater_equal`,
  `M_always`) also used by `DepthTestAttrib`/`StencilAttrib`.

## API

| Method | Notes |
|---|---|
| `static CPT(RenderAttrib) make(PandaCompareFunc mode, PN_stdfloat reference_alpha)` | Construct/intern |
| `static CPT(RenderAttrib) make_default()` | `M_always`, ref `1.0` (test disabled) |
| `PandaCompareFunc get_mode() const` | |
| `PN_stdfloat get_reference_alpha() const` | |

## Usage
```cpp
// Discard pixels with alpha < 0.5
node_path.set_attrib(AlphaTestAttrib::make(AlphaTestAttrib::M_greater_equal, 0.5f));
```

## See also
[RenderAttrib](RenderAttrib.md), [pgraph README — the state pipeline](README.md#the-state-pipeline)
