# PGFrameStyle

**Source:** `panda/src/pgui/pgFrameStyle.h` / `.I` / `.cxx`
**Inherits:** none (standalone value type, not a `PandaNode`)

Describes how to procedurally generate the background/border geometry for one
state of a [PGItem](PGItem.md) (or the highlighted portion of a
[PGWaitBar](PGWaitBar.md)). Cheap to copy; typically built on the stack and
passed to `PGItem::set_frame_style(state, style)`.

## Behavior notes

- **`Type` determines both the geometry and which fields matter.** `T_none`
  generates nothing. `T_flat` is a plain quad (ignores `width`). `T_bevel_out`
  / `T_bevel_in` / `T_groove` / `T_ridge` all use `width` as the border
  thickness and shade the border's four edges with color multipliers to fake a
  3-D bevel (in vs. out / groove vs. ridge just flip which edges are
  light/dark). `T_texture_border` renders a 9-slice-style border split into
  quadrants using `width` (for geometry) and `uv_width` (for how much of the
  texture is consumed by the border) — the interior between borders is *not*
  filled by this style.
- **`get_internal_frame(frame)`** computes the usable interior of a frame after
  subtracting the border width (nothing subtracted for `T_none`/`T_flat`) and
  applying `visible_scale`. This is what widgets use to compute clip/content
  regions — e.g. [PGScrollFrame](PGScrollFrame.md)'s `remanage()` calls this to
  find the frame's internal rectangle before carving out space for scroll bars.
- **`visible_scale`** shrinks/grows the generated geometry around the frame's
  center without changing the logical frame/clickable region — used e.g. by
  `PGSliderBar::setup_slider()` to draw a thinner track than the full clickable
  height.
- **Alpha in `color`** automatically enables `TransparencyAttrib::M_alpha` on
  the generated geometry.
- **`xform(mat)`** scales `width` by the matrix's X/Z axis lengths (used when a
  `PGItem::xform()` is applied, e.g. from `NodePath::set_scale()`) and returns
  whether the style is affected at all (false for `T_none`/`T_flat`).
- **`generate_into()` is normally called automatically** by `PGItem` when a
  state's frame is (re)built; direct calls are rare, confined to
  low-level/custom widget code.

## API

### Type
```cpp
enum Type { T_none, T_flat, T_bevel_out, T_bevel_in, T_groove, T_ridge, T_texture_border };
```
`void set_type(Type)` / `Type get_type() const`

### Appearance
| Signature | Notes |
|---|---|
| `void set_color(r,g,b,a)` / `set_color(const LColor&)` / `LColor get_color() const` | Dominant color; default opaque white |
| `void set_texture(Texture*)` / `bool has_texture() const` / `Texture *get_texture() const` / `void clear_texture()` | Optional texture applied to the frame |
| `void set_width(x,y)` / `set_width(const LVecBase2&)` / `const LVecBase2 &get_width() const` | Border width in screen units; meaning depends on `Type`. Default `(0.1, 0.1)` |
| `void set_uv_width(u,v)` / `get_uv_width()` | Texture-space border width, used only by `T_texture_border`. Default `(0.1, 0.1)` |
| `void set_visible_scale(x,y)` / `get_visible_scale()` | Scale factor applied to generated geometry only, around the frame center. Default `(1, 1)` |

### Queries / generation
| Signature | Notes |
|---|---|
| `LVecBase4 get_internal_frame(const LVecBase4 &frame) const` | Interior rectangle after border + visible_scale |
| `void output(std::ostream&) const` / `operator<<` | Debug printing |
| `bool xform(const LMatrix4 &mat)` *(public, non-PUBLISHED)* | Applies a transform to border widths; returns whether it mattered |
| `NodePath generate_into(const NodePath &parent, const LVecBase4 &frame, int sort = 0)` *(public, non-PUBLISHED)* | Builds and attaches the geometry; normally called by `PGItem` internals |

## Usage

```cpp
PGFrameStyle style;
style.set_type(PGFrameStyle::T_bevel_out);
style.set_color(0.8f, 0.8f, 0.8f, 1.0f);
style.set_width(0.1f, 0.1f);
item->set_frame_style(PGButton::S_ready, style);
```

## See also

[PGItem.md](PGItem.md) (owner of per-state frame styles) ·
[PGWaitBar.md](PGWaitBar.md) (uses a `PGFrameStyle` for the fill bar, not just
the frame) · [README.md](README.md)
