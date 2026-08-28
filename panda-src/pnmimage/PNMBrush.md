# PNMBrush

**Source:** `panda/src/pnmimage/pnmBrush.h` / `.I` / `.cxx`
**Inherits:** `ReferenceCount`
**Inherited by:** *(several private concrete subclasses defined entirely inside `pnmBrush.cxx` — not part of the public API; obtain instances only through the `make_*()` factories below)*

Controls the shape and color of drawing operations performed by a
[PNMPainter](PNMPainter.md). You don't subclass or construct `PNMBrush`
directly — you get one from a static `make_*()` factory, which internally
picks one of several private brush implementations based on the requested
`BrushEffect`.

## Behavior notes

- **A brush is used two ways**: "smeared" along a border/line (`draw()`,
  called once per line point with a `pixel_scale` antialiasing weight) or
  tiled to fill an interior (`fill()`, called once per scanline with an
  `[xfrom, xto]` range). Both are pure virtual — every concrete brush
  implements both.
- **`BrushEffect` changes what compositing math a pixel/image brush does**,
  independent of whether it's a solid color or an image:
  - `BE_set` — overwrite the destination pixel outright (`set_xel_a()`); a
    pixel brush requires `pixel_scale >= 0.5` to draw at all (no partial-alpha
    antialiasing for a hard "set").
  - `BE_blend` — alpha-composite via `PNMImage::blend()`, with `pixel_scale`
    multiplying the brush color's own alpha.
  - `BE_darken` — per-channel minimum with the existing pixel (`fmin`),
    scaled toward white by `pixel_scale` first (so at `pixel_scale=0` it has
    no darkening effect).
  - `BE_lighten` — per-channel maximum with the existing pixel (`fmax`),
    scaled by `pixel_scale`.
- **`make_transparent()`** returns a brush whose `draw()`/`fill()` are
  literal no-ops — used as a pen to get an unbordered shape, or as a fill to
  get an unfilled (outline-only) shape.
- **`make_spot(color, radius, fuzzy, effect)`** doesn't paint a spot
  analytically — it builds a small square `PNMImage` (side `= ceil(radius*2)`)
  and calls `render_spot()` on it (see [PNMImage](PNMImage.md)) with a
  background chosen per-effect (transparent for set/lighten, `color` at alpha
  0 for blend, opaque white for darken), then wraps that image with
  `make_image()`. `fuzzy` controls whether `render_spot()`'s inner/outer
  radius are equal (hard edge) or the inner radius is 0 (soft falloff).
  This means a spot brush is really an image brush under the hood.
- **`make_image()` copies the image** — safe to modify or destroy the source
  `PNMImage` right after the call. For fill, the image brush tiles by
  computing a modulo offset (`(x+xo) % image.get_x_size()`) so multiple
  `PNMPainter`s sharing an `(xo, yo)` origin get aligned tiling.
- **`get_xc()`/`get_yc()`** give the brush's center-pixel coordinate in the
  brush's own local space (0.5, 0.5 for a 1-pixel brush; 1.0, 1.0 for a
  centered 2-pixel brush, etc.) — [PNMPainter](PNMPainter.md) uses this to
  offset line coordinates so the pen is centered on the line, not offset by
  half a pixel.

## API reference

| Signature | Notes |
|---|---|
| `enum BrushEffect { BE_set, BE_blend, BE_darken, BE_lighten }` | See "Behavior notes" for the compositing math of each |
| `static PT(PNMBrush) make_transparent()` | Paints nothing; use for borderless/unfilled shapes |
| `static PT(PNMBrush) make_pixel(const LColorf &color, BrushEffect effect = BE_blend)` | Single-color brush |
| `static PT(PNMBrush) make_spot(const LColorf &color, float radius, bool fuzzy, BrushEffect effect = BE_blend)` | Round spot; `fuzzy` = soft-edged falloff vs. hard edge |
| `static PT(PNMBrush) make_image(const PNMImage &image, float xc, float yc, BrushEffect effect = BE_blend)` | Arbitrary bitmap brush; `xc`/`yc` mark its center pixel; image is copied |
| `float get_xc() const` / `float get_yc() const` | Center-pixel coordinates |
| `virtual void draw(PNMImage &image, int x, int y, float pixel_scale) = 0` | Smear one antialiased point (protected-use, called by `PNMPainter`) |
| `virtual void fill(PNMImage &image, int xfrom, int xto, int y, int xo, int yo) = 0` | Tile-fill one scanline range |

## Usage

```cpp
PT(PNMBrush) pen  = PNMBrush::make_pixel(LColorf(0, 0, 0, 1), PNMBrush::BE_set);
PT(PNMBrush) fill = PNMBrush::make_spot(LColorf(1, 0, 0, 1), 8.0f, /*fuzzy*/ true);

PNMPainter painter(image);
painter.set_pen(pen);    // painter takes ownership
painter.set_fill(fill);
painter.draw_rectangle(10, 10, 50, 40);
```

## See also

[PNMPainter](PNMPainter.md) (the drawing tool that uses a brush as pen/fill) ·
[PNMImage](PNMImage.md) (`render_spot()`, `blend()`, `set_xel_a()` used
internally; the surface every brush ultimately paints onto) ·
[README.md](README.md)
