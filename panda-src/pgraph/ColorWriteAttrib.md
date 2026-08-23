# ColorWriteAttrib

**Source:** `panda/src/pgraph/colorWriteAttrib.h` (+ `.I`, `.cxx`)
**Inherits:** [RenderAttrib](RenderAttrib.md)

Enables or disables writing to specific color-buffer channels — useful for
effects that need to write to the depth buffer without touching color
(e.g. a pre-pass), or writing to only some channels.

## Behavior notes
- `Channels` bit values deliberately match `D3DCOLORWRITEENABLE_RED/
  GREEN/BLUE/ALPHA` (source comment).
- `make_default()` uses the parameterless constructor, which defaults to
  `C_all` — writing all channels is the default/identity state.

## API

| Method | Notes |
|---|---|
| `static CPT(RenderAttrib) make(unsigned int channels)` | Bitwise-OR of `Channels` |
| `static CPT(RenderAttrib) make_default()` | `C_all` |
| `unsigned int get_channels() const` | |

`Channels` enum: `C_off = 0x0`, `C_red = 0x1`, `C_green = 0x2`,
`C_blue = 0x4`, `C_rgb = 0x7`, `C_alpha = 0x8`, `C_all = 0xf`.

## Usage
```cpp
// Write depth only, no color (typical z-prepass)
node_path.set_attrib(ColorWriteAttrib::make(ColorWriteAttrib::C_off));
```

## See also
[RenderAttrib](RenderAttrib.md), [pgraph README — the state pipeline](README.md#the-state-pipeline)
