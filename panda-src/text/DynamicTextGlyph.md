# DynamicTextGlyph

**Source:** `panda/src/text/dynamicTextGlyph.h` (+ `.I`, `.cxx`)
**Inherits:** [TextGlyph](TextGlyph.md)
**Compiled only when `HAVE_FREETYPE` is defined.**

A [TextGlyph](TextGlyph.md) as produced by [DynamicTextFont](DynamicTextFont.md):
adds the bookkeeping for where the glyph's pixels live within a
[DynamicTextPage](DynamicTextPage.md) texture.

## Behavior notes

- **Non-copyable.** Copy constructor and copy assignment are explicitly
  `= delete`d — a `DynamicTextGlyph` owns a specific rectangle within a
  specific page, so copying one wouldn't make sense.
- **Two constructors correspond to two different kinds of glyph.** The full
  constructor (`page`, `x`, `y`, `x_size`, `y_size`, `margin`) is a real,
  visible glyph slotted into a texture page. The two-argument constructor
  (`character`, `advance`) makes an "empty" glyph with `_page == nullptr` —
  used for whitespace or a zero-size FreeType bitmap; `is_whitespace()`
  returns `true` exactly when `_page == nullptr`.
- **`get_row(y)` does the pixel-offset math to hand back a raw pointer into
  the page's RAM image, inverting Y** (`y = page_y_size - 1 - y`) because
  FreeType bitmaps are top-down while the texture's RAM image is bottom-up.
  Only valid between construction and whenever the page's RAM image is
  released — callers (`DynamicTextFont::copy_bitmap_to_texture()` etc.) use
  it during initial glyph rendering only.
- **`erase()` doesn't deallocate the rectangle, it just repaints it.**
  `DynamicTextPage`'s free-space search (`find_hole()`/`find_overlap()`) is a
  linear scan over still-registered `DynamicTextGlyph` objects, not a real
  allocator; a glyph is actually freed by being dropped from the page's
  `_glyphs` vector (in `DynamicTextPage::garbage_collect()`), at which point
  `erase()` is called first to blank its pixels back to the font's background
  color so stale glyph data doesn't leak into whatever gets slotted there
  next.
- **`intersects()` is a plain axis-aligned rectangle overlap test** against
  `(_x, _y, _x_size, _y_size)` — this is the primitive `find_hole()` in
  [DynamicTextPage](DynamicTextPage.md) uses to scan for free space.

## API

| Signature | Notes |
|---|---|
| `DynamicTextGlyph(int character, DynamicTextPage *page, int x, int y, int x_size, int y_size, int margin, PN_stdfloat advance)` | Real, page-backed glyph |
| `DynamicTextGlyph(int character, PN_stdfloat advance)` | Empty/whitespace glyph, no page |
| `DynamicTextPage *get_page() const` | `nullptr` for whitespace glyphs |
| `bool intersects(int x, int y, int x_size, int y_size) const` | Overlap test used by page allocation |
| `PN_stdfloat get_left/bottom/right/top() const` | Vertex-space quad coordinates (`_quad_dimensions`, inherited) |
| `PN_stdfloat get_uv_left/bottom/right/top() const` | UV-space quad coordinates (`_quad_texcoords`, inherited) |
| `unsigned char *get_row(int y)` | Raw pixel row pointer into the page; rendering-time use only |
| `void erase(DynamicTextFont *font)` | Repaints this glyph's pixels with `font->get_bg()` |
| `virtual bool is_whitespace() const` | `true` iff `_page == nullptr` |

## See also

[TextGlyph.md](TextGlyph.md) · [DynamicTextPage.md](DynamicTextPage.md) ·
[DynamicTextFont.md](DynamicTextFont.md#garbage-collection) · [README.md](README.md)
