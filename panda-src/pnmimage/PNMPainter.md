# PNMPainter

**Source:** `panda/src/pnmimage/pnmPainter.h` / `.I` / `.cxx`
**Inherits:** *(none)*
**Inherited by:** *(none)*

Draws lines and filled rectangles directly into a [PNMImage](PNMImage.md),
using a pair of [PNMBrush](PNMBrush.md)es — one for outlines/points ("pen"),
one for interiors ("fill").

## Behavior notes

- **Does not own the target image.** The constructor stores a plain reference
  to the `PNMImage` you pass it; the caller must keep that image alive for the
  painter's whole lifetime. It *does* take ownership of the pen/fill brushes
  passed to `set_pen()`/`set_fill()` — no need to hold a separate `PT()` to
  them.
- **Default pen/fill**: constructed with an opaque-black `BE_blend` pixel pen
  and an opaque-white `BE_blend` pixel fill (`make_pixel(LColorf(0,0,0,1))`
  and `make_pixel(LColorf(1,1,1,1))` respectively) — usable immediately
  without calling `set_pen()`/`set_fill()` first.
- **`draw_point()` is literally `draw_line(x, y, x, y)`** — a degenerate line,
  handled as a special case inside `draw_line()` (treated as a very short
  horizontal segment) rather than as separate logic.
- **`draw_line()` is antialiased via per-pixel coverage weighting, not a
  simple Bresenham walk.** It picks the more nearly horizontal or vertical
  axis and steps pixel-by-pixel along it, calling `draw_hline_point()`/
  `draw_vline_point()` for each column/row crossed; each of those computes
  the fractional cross-axis position and, when it lands between two pixels,
  calls `pen->draw()` on *both* neighbors with complementary fractional
  `pixel_scale` weights (so a `BE_blend` pen produces a smooth antialiased
  edge, but `BE_set` — which requires `pixel_scale >= 0.5` — will only ever
  paint the pixel that's more than half-covered).
- **The pen is recentered using `get_xc()`/`get_yc()`** before any line math —
  `draw_line()` shifts both endpoints by `pen->get_xc() - 0.5` /
  `get_yc() - 0.5` so a multi-pixel pen (e.g. a brush image) is centered on
  the line rather than offset by its origin corner.
- **`draw_rectangle()` fills before it outlines** — the interior is filled
  scanline-by-scanline via `_fill->fill()` over the *inclusive integer* pixel
  range (`ceil(xa)..floor(xb)`, `ceil(ya)..floor(yb)`), then the four edges
  are stroked afterward with four `draw_line()` calls using the pen — so a
  fill brush and a pen brush can visibly overlap at the border by design
  (the outline is drawn on top of the fill).
- **`(xo, yo)` constructor offset only affects fill-pattern alignment** — it's
  passed straight through to `brush->fill()`'s `xo`/`yo` parameters, letting a
  tiled image-brush fill line up with a logical "virtual canvas" origin when
  painting into a sub-region of a larger conceptual image (see
  [PNMBrush](PNMBrush.md)'s tiling behavior notes).

## API reference

| Signature | Notes |
|---|---|
| `explicit PNMPainter(PNMImage &image, int xo = 0, int yo = 0)` | Does not take ownership of `image`; `xo`/`yo` offset fill-pattern tiling only |
| `void set_pen(PNMBrush *pen)` / `PNMBrush *get_pen() const` | Painter takes ownership of `pen` |
| `void set_fill(PNMBrush *fill)` / `PNMBrush *get_fill() const` | Painter takes ownership of `fill` |
| `void draw_point(float x, float y)` | Equivalent to `draw_line(x, y, x, y)` |
| `void draw_line(float xa, float ya, float xb, float yb)` | Antialiased, pen-centered |
| `void draw_rectangle(float xa, float ya, float xb, float yb)` | Corners in either order; fills interior then strokes outline |

## Usage

```cpp
PNMImage image(256, 256, 4);
image.fill(1, 1, 1);
image.alpha_fill(1.0f);

PNMPainter painter(image);
painter.set_pen(PNMBrush::make_pixel(LColorf(0, 0, 0, 1)));
painter.set_fill(PNMBrush::make_pixel(LColorf(1, 0, 0, 1)));
painter.draw_rectangle(20, 20, 100, 80);
painter.draw_line(0, 0, 255, 255);
```

## See also

[PNMBrush](PNMBrush.md) (pen/fill implementations, `BrushEffect` semantics) ·
[PNMImage](PNMImage.md) (the surface painted into) · [README.md](README.md)
