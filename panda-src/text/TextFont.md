# TextFont

**Source:** `panda/src/text/textFont.h` (+ `.I`, `.cxx`)
**Inherits:** `TypedReferenceCount`, `Namable`
**Inherited by:** [StaticTextFont](StaticTextFont.md), [DynamicTextFont](DynamicTextFont.md)

Abstract interface for "a set of glyphs that may be assembled together by a
`TextNode` to represent a string of text." Never instantiated directly — use
[StaticTextFont](StaticTextFont.md) or [DynamicTextFont](DynamicTextFont.md),
or load one through [FontPool](FontPool.md).

## Behavior notes

- **`get_glyph(character)`, the public inline version, always returns
  *something*.** It calls the pure-virtual `get_glyph(character, glyph)`
  overload; if that fails to find a real glyph, the glyph reference may still
  come back non-null (an "invalid glyph" placeholder) even though the
  character wasn't actually in the font.
- **The invalid-glyph placeholder is a hollow rectangle outline, built lazily
  once per font.** `get_invalid_glyph()` calls `make_invalid_glyph()` on first
  use, which constructs a `GeomLinestrips` box sized `0.2..0.5 × line_height`
  wide and `0.1..0.7 × line_height` tall — not an empty/whitespace glyph, so
  missing characters are visually obvious.
- **`get_kerning()` defaults to always returning 0.** Only `DynamicTextFont`
  overrides it (using FreeType's kerning tables); `StaticTextFont` never
  supports kerning.
- **Default construction values:** `_line_height = 1.0`, `_space_advance =
  0.25`, `_is_valid = false`. Subclass constructors are responsible for
  setting `_is_valid = true` and computing real `_line_height`/`_space_advance`
  values from the underlying font data.
- **`RenderMode` is texture-based by default (`RM_texture`); most other modes
  only apply to `DynamicTextFont`.** `StaticTextFont` glyphs come from a model
  file and ignore `RenderMode` entirely.

## API

### Validity & metrics
| Signature | Notes |
|---|---|
| `bool is_valid() const` / `operator bool() const` | Whether the font loaded successfully |
| `PN_stdfloat get_line_height() const` / `set_line_height(PN_stdfloat)` | Vertical distance between lines |
| `PN_stdfloat get_space_advance() const` / `set_space_advance(PN_stdfloat)` | Width of a space character |
| `PN_stdfloat get_total_poly_margin() const` | Margin between a glyph's edge and its card edge (protected setter, subclass-only) |

### Glyphs
| Signature | Notes |
|---|---|
| `CPT(TextGlyph) get_glyph(int character)` | Convenience wrapper; may return the invalid-glyph placeholder |
| `virtual bool get_glyph(int character, CPT(TextGlyph) &glyph) = 0` | The real lookup; returns `false` if the character wasn't found (glyph may still be filled in) |
| `TextGlyph *get_invalid_glyph()` | Lazily-built placeholder glyph for missing characters |
| `virtual PN_stdfloat get_kerning(int first, int second) const` | Extra offset between two adjacent glyphs; base class always returns 0 |

### RenderMode
```cpp
enum RenderMode {
  RM_texture, RM_wireframe, RM_polygon, RM_extruded, RM_solid,
  RM_distance_field, RM_invalid,
};
static RenderMode string_render_mode(const std::string &string);
```
`string_render_mode()` does case-insensitive matching against `"texture"`,
`"wireframe"`, `"polygon"`, `"extruded"`, `"solid"`, `"distance_field"`;
anything else returns `RM_invalid`. `operator<<`/`operator>>` round-trip these
names for `Config.prc` parsing (`text-render-mode`).

### Other
| Signature | Notes |
|---|---|
| `virtual PT(TextFont) make_copy() const = 0` | Deep-ish copy (subclass-defined) |
| `virtual void write(std::ostream&, int indent_level) const` | Base prints just the name; subclasses add glyph listings |

## Usage

```cpp
PT(TextFont) font = FontPool::load_font("myfont.ttf");
if (font != nullptr && font->is_valid()) {
  text_node->set_font(font);
}
```

## See also

[StaticTextFont.md](StaticTextFont.md) · [DynamicTextFont.md](DynamicTextFont.md)
· [TextGlyph.md](TextGlyph.md) · [FontPool.md](FontPool.md) · [README.md](README.md)
