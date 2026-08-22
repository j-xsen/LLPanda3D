# DynamicTextPage

**Source:** `panda/src/text/dynamicTextPage.h` (+ `.I`, `.cxx`)
**Inherits:** `Texture`
**Compiled only when `HAVE_FREETYPE` is defined.**

One texture atlas "page" belonging to a [DynamicTextFont](DynamicTextFont.md).
A font starts with zero pages and adds more as needed
(`DynamicTextFont::slot_glyph()`); each page is a real `Texture` that can be
bound and rendered like any other.

## Behavior notes

- **Constructed pre-filled with the font's background color.** The
  constructor sizes the texture from `font->get_page_size()`, disables
  compression (texture content changes at runtime), copies the font's
  quality-level/filter/anisotropic-degree settings, sets an explicit
  invisible border color (`text-wrap-mode`, defaulting to
  `WM_border_color`) to avoid edge bleeding, and immediately calls
  `fill_region()` to paint the whole texture with `font->get_bg()` before any
  glyph is ever slotted in.
- **`set_keep_ram_image(true)` is set unconditionally** — the page's RAM
  image must never be discarded, since glyphs are drawn into it directly on
  the CPU side (`get_row()`/`fill_region()`) at arbitrary times after initial
  upload, not just once at load time.
- **Glyph placement is first-fit via linear scan, not a real packer.**
  `find_hole(x, y, x_size, y_size)` scans row by row: for each `y`, it scans
  `x` rightward, and whenever it hits an existing glyph's rectangle
  (`find_overlap()`), it jumps `x` past that glyph's right edge and clamps
  `next_y` to that glyph's bottom edge, so the next row scanned is the
  earliest row where *something* changes. This is simple and reasonably
  cache-friendly for left-to-right glyph insertion order, but it is not a
  shelf/skyline packer and can waste space with pathological insertion
  patterns.
- **`fill_region()` has a hand-unrolled fast path per pixel format**
  (1/2/3/4-component), writing raw bytes/`uint16_t`/`uint32_t` directly into
  `modify_ram_image()` rather than going through a generic per-pixel API —
  this is a hot path called once per page construction and once per glyph
  erased during garbage collection.
- **`garbage_collect(font)` must be called from
  `DynamicTextFont::garbage_collect()`, never standalone** — the header
  comment for this method is explicit that the font's own glyph index must be
  cleared first, since this method only manages the *page's* `_glyphs` list
  and calls `glyph->erase(font)` on anything it drops; calling it out of that
  order would leave the font's cache holding dangling/incorrect glyph
  pointers.
- **`is_empty()` (no glyphs at all) is used by `slot_glyph()`'s page-rotation
  search as a "give up" signal** — if a completely empty page still can't fit
  a requested glyph size, no other page will be able to either (an empty page
  is the largest possible contiguous free space), so the search aborts rather
  than continuing to check every remaining page.

## API

| Signature | Notes |
|---|---|
| `DynamicTextPage(DynamicTextFont *font, int page_number)` | See construction notes above |
| `const LVecBase2i &get_size() const` / `get_x_size()` / `get_y_size()` | Texture dimensions in pixels |
| `bool is_empty() const` | No glyphs slotted on this page |
| `DynamicTextGlyph *slot_glyph(int character, int x_size, int y_size, int margin, PN_stdfloat advance)` | Finds space via `find_hole()`; returns `nullptr` if it doesn't fit |
| `void fill_region(int x, int y, int x_size, int y_size, const LColor&)` | Direct RAM-image paint, format-specialized |

## See also

[DynamicTextFont.md](DynamicTextFont.md#garbage-collection) ·
[DynamicTextGlyph.md](DynamicTextGlyph.md) · [README.md](README.md)
