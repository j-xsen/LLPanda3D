# TextGlyph

**Source:** `panda/src/text/textGlyph.h` (+ `.I`, `.cxx`)
**Inherits:** `TypedReferenceCount`
**Inherited by:** [DynamicTextGlyph](DynamicTextGlyph.md)

One renderable character from a font: either a simple textured quad, or
arbitrary `Geom` geometry, plus an advance width. `StaticTextFont` constructs
plain `TextGlyph`s directly; `DynamicTextFont` constructs the
[DynamicTextGlyph](DynamicTextGlyph.md) subclass.

## Behavior notes

- **A glyph can represent its shape two ways, and `has_quad()` distinguishes
  them.** A "quad" glyph (`set_quad()`) stores just dimensions + UVs and is
  cheap to batch (see [TextAssembler](TextAssembler.md#quads-vs-arbitrary-geoms)'s
  quad-merging path). A "Geom" glyph (`set_geom()`) stores arbitrary
  vertex/primitive data (used for wireframe/polygon/extruded render modes, or
  any `StaticTextFont` glyph whose source geometry isn't a clean quad).
  Setting one clears the other.
- **`get_geom()` always returns a fresh copy, wrapped as a
  [GeomTextGlyph](GeomTextGlyph.md)** — needed for ref-count-based garbage
  collection to work across `Geom` copies/merges (see
  [README.md's core concepts](README.md#core-concepts) and
  [GeomTextGlyph.md](GeomTextGlyph.md) for why). If a quad glyph has no
  cached `_geom` yet, `get_geom()` synthesizes one from `_quad_dimensions`/
  `_quad_texcoords` on first call (`make_quad_geom()`) and caches it.
- **`check_quad_geom()` auto-detects whether an arbitrary loaded `Geom` is
  secretly a simple textured quad**, so `StaticTextFont` glyphs loaded from an
  `egg-mkfont` model can still benefit from quad batching. It requires exactly
  4 vertices, 1 indexed triangle-decomposable primitive with 6 vertex
  references (i.e. 2 triangles), at most a vertex + texcoord column, all
  vertices at `y == 0`, and the vertices/texcoords forming an axis-aligned
  rectangle — anything more exotic (a colored quad, a non-rectangular shape)
  falls back to being treated as arbitrary geometry.
- **`is_whitespace()` on the base class always returns `false`.** A plain
  `TextGlyph` (as used by `StaticTextFont`) has no notion of whitespace —
  every character in the model file is visible. Only
  [DynamicTextGlyph](DynamicTextGlyph.md) overrides this to return `true` for
  glyphs with no page (e.g. space).
- **`calc_tight_bounds()` reads straight from `_quad_dimensions` for quad
  glyphs** (no need to walk vertex data), falling back to `Geom::calc_tight_bounds()`
  otherwise.

## API

### Construction (protected/inline — subclasses and font code only)
| Signature | Notes |
|---|---|
| `TextGlyph(int character, PN_stdfloat advance = 0)` | Empty glyph, only remembers its advance |
| `TextGlyph(int character, const Geom *geom, const RenderState *state, PN_stdfloat advance)` | Full constructor; auto-runs `check_quad_geom()` if `geom` is non-null |

### Query
| Signature | Notes |
|---|---|
| `int get_character() const` | Unicode code point this glyph represents |
| `bool has_quad() const` | True if representable as a single textured rectangle |
| `bool get_quad(LVecBase4 &dimensions, LVecBase4 &texcoords) const` | Fails if `!has_quad()`; order is left, bottom, right, top |
| `const RenderState *get_state() const` | Render state (usually the font's texture + transparency attribs) |
| `PN_stdfloat get_advance() const` | Distance to advance the pen after this character |
| `virtual bool is_whitespace() const` | `false` on the base class always |
| `PT(Geom) get_geom(Geom::UsageHint) const` | Always a fresh copy, wrapped as `GeomTextGlyph` |

### Mutation (font-construction code only)
| Signature | Notes |
|---|---|
| `void set_quad(const LVecBase4 &dimensions, const LVecBase4 &texcoords, const RenderState *state)` | Clears any `Geom`; sets `has_quad() == true` |
| `void set_geom(GeomVertexData*, GeomPrimitive*, const RenderState*)` | Clears quad state; wraps the result as a `GeomTextGlyph` referencing `this` |
| `void calc_tight_bounds(LPoint3 &min, LPoint3 &max, bool &found_any, Thread*) const` | Used by cheesy-accent positioning in `TextAssembler` |

## Usage

```cpp
// Typical read-only usage, from font/glyph-lookup code:
CPT(TextGlyph) glyph;
if (font->get_glyph('A', glyph)) {
  PN_stdfloat width = glyph->get_advance();
  LVecBase4 dims, uvs;
  if (glyph->get_quad(dims, uvs)) {
    // fast path: batch as a quad
  } else {
    PT(Geom) geom = glyph->get_geom(Geom::UH_static);
    // arbitrary geometry path
  }
}
```

## See also

[DynamicTextGlyph.md](DynamicTextGlyph.md) · [GeomTextGlyph.md](GeomTextGlyph.md)
· [TextFont.md](TextFont.md) · [TextAssembler.md](TextAssembler.md) · [README.md](README.md)
