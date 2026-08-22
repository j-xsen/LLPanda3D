# DynamicTextFont

**Source:** `panda/src/text/dynamicTextFont.h` (+ `.I`, `.cxx`)
**Inherits:** [TextFont](TextFont.md), `FreetypeFont` (`panda/src/pnmtext`)
**Compiled only when `HAVE_FREETYPE` is defined.**

A `TextFont` that rasterizes glyphs from a standard font file (TTF, etc.) on
demand, via FreeType, caching each rendered glyph into a
[DynamicTextPage](DynamicTextPage.md) texture atlas. This is the font type
used for essentially all real-world text rendering in Panda3D; `StaticTextFont`
exists mainly as the no-FreeType fallback.

## Behavior notes

- **Glyphs are cached by FreeType glyph index, not by character code.**
  `get_glyph()` maps `character → FT_Get_Char_Index()` first, then looks up
  `_cache` by that glyph index — so two different Unicode code points that
  the font maps to the same glyph (common with font-internal aliasing) share
  one cached `TextGlyph`/texture-page slot.
- **Configuration setters that affect existing texture layout assert
  `get_num_pages() == 0`.** `set_point_size()`, `set_pixels_per_unit()`,
  `set_scale_factor()`, `set_native_antialias()`, `set_fg()`, `set_bg()`, and
  `set_outline()` all `nassert`-guard on no pages existing yet — i.e. no
  characters have been rendered yet, or `clear()` was just called. Calling
  these after glyphs have already been rasterized is a programming error (in
  a release build the assert is a no-op and the call silently does nothing
  useful, since existing glyphs on existing pages are never touched).
- **Texture format (`_tex_format`) is derived from fg/bg/outline colors, and
  fast-pathed for the common case.** `determine_tex_format()` inspects
  whether any of fg/bg/outline carry actual RGB color vs. pure grayscale,
  and whether any carry non-opaque alpha, picking `F_rgba`/`F_rgb`/
  `F_luminance_alpha`/`F_luminance`/`F_alpha` accordingly. The default
  fg=opaque-white / bg=transparent-white / no-outline combination sets
  `_needs_image_processing = false`, letting `make_glyph()` memcpy FreeType's
  raw bitmap straight into the texture instead of going through the slower
  PNMImage blend path — so leaving colors at their defaults (and coloring text
  via `TextProperties::set_text_color()` at render time instead) is
  meaningfully cheaper.
- **`RM_texture` is not the only render mode, but is the only one that uses a
  texture at all.** `RM_wireframe`/`RM_polygon`/`RM_extruded`/`RM_solid` ask
  FreeType to `decompose_outline()` the glyph into vector contours and build
  actual 3-D line/triangle geometry (`render_wireframe_contours()`/
  `render_polygon_contours()`, using a `Triangulator` for the solid faces,
  with holes handled for e.g. the counter of an "o"). `RM_distance_field`
  computes a padded grayscale distance-field bitmap in a `PNMImage`
  (`render_distance_field()`, from `FreetypeFont`) rather than FreeType's
  normal glyph rasterization.
- **Outline rendering is a two-pass Gaussian blur trick, not real geometric
  outlining.** `copy_pnmimage_to_texture()` first Gaussian-blurs the glyph
  bitmap by (roughly) the outline width to approximate an outline shape, then
  remaps the blurred alpha through a squared `_outline_feather` curve to
  harden the edge, blends that in the outline color, and *then* blends the
  original sharp glyph on top in the foreground color.
- **`get_kerning()` divides FreeType's kerning delta by
  `_font_pixels_per_unit * 64`** — FreeType kerning values are in 26.6
  fixed-point font units; `text-kerning` must be enabled and the font must
  report `FT_HAS_KERNING` or this always returns 0.
- **The compiled-in default font sets a non-default winding order.**
  `TextProperties::load_default_font()` explicitly calls
  `set_winding_order(WO_left)` on the compiled-in FreeType font — noted here
  because it's a real, source-visible quirk of that one font file, not a
  general `DynamicTextFont` default.

### Garbage collection

- **`garbage_collect()` operates in two passes: the font's index, then each
  page.** First it rebuilds `_cache`, dropping any entry whose `TextGlyph`
  has `get_ref_count() <= 1` (meaning only the cache itself references it —
  see [GeomTextGlyph](GeomTextGlyph.md) for how that ref count stays accurate
  through `Geom` merging). Then it calls `DynamicTextPage::garbage_collect()`
  on every page, which does the equivalent pass over its own `_glyphs` list
  and additionally **erases the freed glyph's pixels** from the texture (fills
  with the font's background color) so the space can be reused by
  `slot_glyph()`.
- **`clear()` is much blunter than `garbage_collect()`** — it drops the
  *entire* cache and page list unconditionally, regardless of whether glyphs
  are still rendered somewhere. Any already-rendered text keeps working (it
  holds its own references to the old pages/glyphs), but calling this
  frequently wastes texture memory by orphaning pages that stay alive only
  because of old geometry.
- **New pages are only created after a garbage-collect attempt fails.**
  `slot_glyph()` searches existing pages starting from `_preferred_page`
  (rotating forward to spread allocations, and remembering whichever page
  last succeeded as the new preferred page); only if *no* existing page has
  room does it call `garbage_collect()` and retry once recursively before
  finally allocating a brand new `DynamicTextPage`.

## API

### Construction
| Signature | Notes |
|---|---|
| `DynamicTextFont(const Filename &font_filename, int face_index = 0)` | Load from a file on disk |
| `DynamicTextFont(const char *font_data, int data_length, int face_index)` | Load from an in-memory buffer (used for the compiled-in default font) |
| `virtual PT(TextFont) make_copy() const` | Copies FreeType face settings; starts with zero pages/cache (independent glyph rendering) |

### Rendering parameters (assert `get_num_pages() == 0`, see notes)
| Signature | Notes |
|---|---|
| `bool set_point_size(PN_stdfloat)` / `get_point_size()` | Font size; ~10pt ≈ 1 screen unit by convention |
| `bool set_pixels_per_unit(PN_stdfloat)` / `get_pixels_per_unit()` | Texture resolution per screen unit |
| `bool set_scale_factor(PN_stdfloat)` / `get_scale_factor()` | FreeType oversample-then-downfilter factor, for antialiasing |
| `void set_native_antialias(bool)` / `get_native_antialias()` | FreeType's own antialiasing, independent of `scale_factor` |
| `int get_font_pixel_size() const` | Nonzero only if approximating with a fixed-pixel-size (non-scalable) font |
| `void set_fg/get_fg(const LColor&)`, `set_bg/get_bg`, `set_outline(color, width, feather)` + getters | Colors baked into the rendered texture (see notes on cost) |
| `Texture::Format get_tex_format() const` | Derived automatically by `determine_tex_format()` |
| `void set_render_mode(RenderMode)` / `get_render_mode()` | See `RenderMode` note above |

### Texture / page parameters
| Signature | Notes |
|---|---|
| `void set_texture_margin(int)` / `get_texture_margin()` | Texel padding around each glyph (default `text-texture-margin`) |
| `void set_poly_margin(PN_stdfloat)` / `get_poly_margin()` | Geometric padding around each glyph's quad |
| `void set_page_size(int x, int y)` / `get_page_size()` / `get_page_x_size()` / `get_page_y_size()` | Texture dimensions for new pages |
| `void set_minfilter/magfilter(SamplerState::FilterType)` + getters | Applied to all existing + future pages via `update_filters()` |
| `void set_anisotropic_degree(int)` / `get_anisotropic_degree()` | Same |

### Pages & cache
| Signature | Notes |
|---|---|
| `int get_num_pages() const` / `DynamicTextPage *get_page(int n) const` / `MAKE_SEQ(get_pages, ...)` | Enumerate texture pages |
| `int garbage_collect()` | See notes above; returns glyphs removed from the font's index |
| `void clear()` | Drops everything; see notes above |

### Lookup
| Signature | Notes |
|---|---|
| `virtual bool get_glyph(int character, CPT(TextGlyph)&)` | Caches by FreeType glyph index |
| `bool get_glyph_by_index(int character, int glyph_index, CPT(TextGlyph)&)` | Used by the HarfBuzz shaping path, which already has glyph indices |
| `virtual PN_stdfloat get_kerning(int first, int second) const` | See notes above |
| `hb_font_t *get_hb_font() const` | Lazily creates (and owns) a HarfBuzz font wrapper; `nullptr` if `!HAVE_HARFBUZZ` |

## Usage

```cpp
PT(DynamicTextFont) font = new DynamicTextFont("myfont.ttf");
font->set_page_size(512, 512);      // must precede any glyph rendering
font->set_pixels_per_unit(60);
text_node->set_font(font);
```

## See also

[TextFont.md](TextFont.md) · [DynamicTextGlyph.md](DynamicTextGlyph.md) ·
[DynamicTextPage.md](DynamicTextPage.md) · [GeomTextGlyph.md](GeomTextGlyph.md)
· [FontPool.md](FontPool.md) · [TextAssembler.md](TextAssembler.md#harfbuzz-shaping)
· [README.md](README.md)
