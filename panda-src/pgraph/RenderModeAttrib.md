# RenderModeAttrib

**Source:** `panda/src/pgraph/renderModeAttrib.h` (+ `.I`, `.cxx`)
**Inherits:** RenderAttrib

Specifies how polygons at and below this node are rasterized — filled,
wireframe, points-only, or filled-with-wireframe-overlay.

## Behavior notes

- `M_unchanged` is a pass-through mode: `compose_impl()` treats a
  downstream `M_unchanged` attrib as "keep whatever mode was already in
  effect" rather than overriding it — the only `RenderAttrib` subclass in
  this batch with non-default `compose_impl()` behavior.
- `thickness` doubles as line width (wireframe/linestrip) and point
  diameter; `perspective` switches point sizing between screen-space pixels
  (default) and 3-d world units that scale with distance.
- `M_filled_wireframe` draws filled polygons with a flat-colored wireframe
  overlay in `wireframe_color`; that color is only serialized to Bam files
  when `_mode == M_filled_wireframe`.
- `get_geom_rendering(int)` folds in the `Geom::GeomRendering` bits this
  mode requires (`GR_point`/`GR_render_mode_point` for `M_point`,
  `GR_render_mode_wireframe` for `M_wireframe`, plus perspective/uniform-size
  point bits) — used by the renderer to decide how to expand geometry.

## API

| Signature | Notes |
|---|---|
| `enum Mode` | `M_unchanged`, `M_filled`, `M_wireframe`, `M_point`, `M_filled_flat` (no perspective correction, useful for software-rendered sprites), `M_filled_wireframe` |
| `static CPT(RenderAttrib) make(Mode mode, PN_stdfloat thickness=1.0f, bool perspective=false, const LColor &wireframe_color=LColor::zero())` | |
| `static CPT(RenderAttrib) make_default()` | `M_filled`, thickness 1, no perspective |
| `Mode get_mode() const` | |
| `PN_stdfloat get_thickness() const` | Line width / point diameter |
| `bool get_perspective() const` | Points scale with distance if true |
| `const LColor &get_wireframe_color() const` | Used in `M_filled_wireframe` |
| `int get_geom_rendering(int geom_rendering) const` | OR's in required `Geom::GeomRendering` bits |

## Usage

```cpp
node_path.set_render_mode_wireframe();                  // NodePath convenience wrapper
node_path.set_attrib(RenderModeAttrib::make(
    RenderModeAttrib::M_filled_wireframe, 1.0f, false, LColor(1, 0, 0, 1)));
```

## See also

[README — the state pipeline](README.md#the-state-pipeline),
[RenderAttrib](RenderAttrib.md)
