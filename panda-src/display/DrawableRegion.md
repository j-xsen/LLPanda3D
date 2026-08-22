# DrawableRegion

**Source:** `panda/src/display/drawableRegion.h` (+ `.I`, `.cxx`)
**Inherits:** (none) **Inherited by:** [GraphicsOutput](GraphicsOutput.md) (also inherits `GraphicsOutputBase`, external), [DisplayRegion](DisplayRegion.md) (also inherits `TypedReferenceCount`)

Shared base of the two "rectangular region you can issue drawing/clear
commands into" concepts in the display module: a whole `GraphicsOutput`
(window or buffer) and a `DisplayRegion` (sub-rectangle within one). Not a
linear inheritance chain — `GraphicsOutput` and `DisplayRegion` each inherit
`DrawableRegion` independently alongside their own primary base. Provides
the clear-color/clear-depth/clear-stencil (and generically, any
`RenderTexturePlane` bitplane) active-flag-plus-value API, plus the
pixel-zoom mechanism. Not a `TypedObject` — it's a plain mixin base with no
`TypeHandle`/`get_class_type()` of its own; `is_of_type`/`DCAST` don't apply
to it directly.

## Behavior notes

- **Clear state is a bitmask (`_clear_mask`) plus a per-plane value array
  (`_clear_value[RTP_COUNT]`).** `set_clear_color_active()` /
  `set_clear_depth_active()` / `set_clear_stencil_active()` are thin inline
  wrappers over the generic virtual `set_clear_active(int n, bool)`, keyed
  by `RTP_color`/`RTP_depth`/`RTP_stencil`. Any of the 16 `RenderTexturePlane`
  values (color, depth, stencil, depth_stencil, 4 aux-rgba, 4 aux-hrgba, 4
  aux-float) can independently have its own clear-active flag and clear
  value — not just the three named convenience methods.
- **Default clear values on construction**: every plane defaults to
  `LColor(0,0,0,0)` except `RTP_depth`, which defaults to `(1,1,1,1)` (i.e.
  clear depth defaults to the far plane, `1.0`). `_pixel_zoom` and
  `_pixel_factor` both default to `1.0`. `_screenshot_buffer_type` defaults
  to `RenderBuffer::T_front`, `_draw_buffer_type` to `RenderBuffer::T_back`
  — i.e. a `DrawableRegion` assumes double-buffering by default; a
  `GraphicsOutput` subclass with a single-buffered `FrameBufferProperties`
  overrides `_draw_buffer_type` to `T_front` in its own constructor (see
  [GraphicsOutput.md](GraphicsOutput.md)).
- **`is_any_clear_active()` is what actually gates whether a clear
  happens** — a `DisplayRegion` or `GraphicsOutput` with every clear-active
  flag false performs no clear at all, leaving whatever was already in the
  buffer. `disable_clears()` zeroes the whole mask in one call.
- **`supports_pixel_zoom()` defaults to `false`** at this base level (each
  concrete GSG backend overrides it); the comment in `set_pixel_zoom()`
  notes the feature is only meaningfully honored by the TinyDisplay
  software renderer — other backends accept the call but silently ignore
  it. `get_pixel_factor()` always reads back as `1.0` when unsupported,
  regardless of what `set_pixel_zoom()` was given.
- **`update_pixel_factor()` recomputes `_pixel_factor` as `1/sqrt(max(zoom,
  1.0))`** — so a zoom of `4.0` yields a pixel factor of `0.5` (half as many
  actual pixels rendered per axis), and any zoom `< 1.0` is clamped to no
  effect. It's called automatically whenever the clear mask or the zoom
  value changes, and calls the virtual `pixel_factor_changed()` hook only
  when the computed factor actually changes — `GraphicsOutput` overrides
  that hook to trigger `set_size_and_recalc()` so the framebuffer's
  effective pixel dimensions track the new zoom (see
  [GraphicsOutput.md](GraphicsOutput.md)'s `get_fb_size()`/`get_fb_x_size()`/
  `get_fb_y_size()`, which apply `get_pixel_factor()` to `get_size()`).
- **`get_renderbuffer_type(int plane)` is the bridge from the "which
  bitplane" enum (`RenderTexturePlane`) to the "which hardware buffer" bitmask
  (`RenderBuffer::Type`)** used when actually issuing GSG copy/clear
  commands. `RTP_depth_stencil` maps to the OR of `T_depth | T_stencil`;
  every other plane (including `RTP_depth`) maps to exactly one
  `RenderBuffer::Type` bit. `GraphicsOutput::copy_to_textures()` calls this
  directly for every plane except `RTP_color`, which instead goes through
  `GraphicsStateGuardian::get_render_buffer()` to also account for
  single/double buffering.
- **Copy constructor/assignment copy every clear value and the zoom
  state**, but `copy_clear_settings()` copies only the clear mask + clear
  values (not draw/screenshot buffer type or zoom) — used by
  `GraphicsOutput::make_cube_map()` to propagate a buffer's clear settings
  onto each per-face `DisplayRegion` it creates.

## API

### Clear-active flags (convenience wrappers over color/depth/stencil)

| Signature |
|---|
| `INLINE void set_clear_color_active(bool)` / `INLINE bool get_clear_color_active() const` |
| `INLINE void set_clear_depth_active(bool)` / `INLINE bool get_clear_depth_active() const` |
| `INLINE void set_clear_stencil_active(bool)` / `INLINE bool get_clear_stencil_active() const` |

### Clear values

| Signature |
|---|
| `INLINE void set_clear_color(const LColor &)` / `INLINE const LColor &get_clear_color() const` |
| `INLINE void set_clear_depth(PN_stdfloat)` / `INLINE PN_stdfloat get_clear_depth() const` |
| `INLINE void set_clear_stencil(unsigned int)` / `INLINE unsigned int get_clear_stencil() const` |

### Generic per-bitplane access

| Signature | Notes |
|---|---|
| `enum RenderTexturePlane { RTP_stencil, RTP_depth_stencil, RTP_color, RTP_aux_rgba_0..3, RTP_aux_hrgba_0..3, RTP_aux_float_0..3, RTP_depth, RTP_COUNT }` | |
| `virtual void set_clear_active(int n, bool)` / `virtual bool get_clear_active(int n) const` | `n` is any `RenderTexturePlane` value. |
| `virtual void set_clear_value(int n, const LColor &)` / `virtual const LColor &get_clear_value(int n) const` | |
| `virtual void disable_clears()` | Zeroes the whole clear mask. |
| `virtual bool is_any_clear_active() const` | Gate for whether a clear happens at all. |
| `static int get_renderbuffer_type(int plane)` | Maps a `RenderTexturePlane` to a `RenderBuffer::Type` bitmask. |
| `INLINE void copy_clear_settings(const DrawableRegion &copy)` | Copies mask + values only, not zoom/buffer-type. |

### Pixel zoom

| Signature | Notes |
|---|---|
| `virtual void set_pixel_zoom(PN_stdfloat)` / `INLINE PN_stdfloat get_pixel_zoom() const` | Set value, regardless of support. |
| `INLINE PN_stdfloat get_pixel_factor() const` | Effective scale actually applied; `1.0` if unsupported. |
| `virtual bool supports_pixel_zoom() const` | Defaults `false`; overridden per-backend (meaningfully only by TinyDisplay). |

### Internal / protected

| Signature | Notes |
|---|---|
| `INLINE int get_screenshot_buffer_type() const` | Default `RenderBuffer::T_front`. |
| `INLINE int get_draw_buffer_type() const` | Default `RenderBuffer::T_back`; `GraphicsOutput` overrides to `T_front` for single-buffered targets. |
| `virtual void pixel_factor_changed()` | Hook called when `update_pixel_factor()` computes a new factor; no-op here, overridden by `GraphicsOutput`. |

## See also

- [GraphicsOutput.md](GraphicsOutput.md) — whole-window/buffer drawable region; the clear settings set here are what `clear()` acts on.
- [DisplayRegion.md](DisplayRegion.md) — sub-rectangle drawable region within a `GraphicsOutput`.
- [README.md](README.md) — module overview, `RenderBuffer`/`GraphicsStateGuardian` cross-references, `background-color` config default.
