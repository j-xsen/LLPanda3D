# AntialiasAttrib

**Source:** `panda/src/pgraph/antialiasAttrib.h` (+ `.I`, `.cxx`)
**Inherits:** [RenderAttrib](RenderAttrib.md)

Specifies whether and how to enable antialiasing (point/line/polygon
smoothing, or framebuffer multisampling), if the backend renderer supports
it.

## Behavior notes
- `_mode` packs a type field (`M_none`, `M_point`, `M_line`, `M_polygon`,
  `M_multisample`, `M_auto`, masked by `M_type_mask`) and a quality-hint
  field (`M_faster`, `M_better`, `M_dont_care`) into one `unsigned short`;
  `get_mode_type()`/`get_mode_quality()` split them back out.
- `M_auto` resolves at render time: prefers `M_multisample` if the hardware
  supports it, else `M_polygon`, unless drawing lines/points (then
  `M_line`/`M_point`, which generally beat multisample for those
  primitives).
- `M_multisample`, when available, ignores `M_point`/`M_line`/`M_polygon`.
  `M_point`/`M_line` smoothing may force transparency on; `M_polygon`
  needs an alpha channel and works best with front-to-back sorted
  primitives.
- `compose_impl()`: if either attrib's type is `M_none` or `M_auto`, the
  lower (later-applied, i.e. `other`) attrib's type wins outright rather
  than unioning; otherwise the two type bitmasks union. Quality bits: the
  lower attrib's quality wins if set, else the upper's is kept.
- `make_default()` doesn't hardcode `M_none` — it looks up the interned
  slot default, which `init_type()` seeds from the config var
  `default-antialias-enable` (bool, default `false`): `M_auto` if true,
  `M_none` if false.

## API

| Method | Notes |
|---|---|
| `static CPT(RenderAttrib) make(unsigned short mode)` | `mode` = type bits `\|` quality bits |
| `static CPT(RenderAttrib) make_default()` | Slot default; see config var above |
| `unsigned short get_mode() const` | Full mode incl. quality bits |
| `unsigned short get_mode_type() const` | Type bits only |
| `unsigned short get_mode_quality() const` | Quality bits only |

`Mode` enum: `M_none`, `M_point`, `M_line`, `M_polygon`, `M_multisample`,
`M_auto`, `M_type_mask` (mask covering the type bits), `M_faster`,
`M_better`, `M_dont_care`.

## Usage
```cpp
node_path.set_attrib(AntialiasAttrib::make(AntialiasAttrib::M_polygon |
                                            AntialiasAttrib::M_better));
```

## See also
[RenderAttrib](RenderAttrib.md), [pgraph README — the state pipeline](README.md#the-state-pipeline)
