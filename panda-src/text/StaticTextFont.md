# StaticTextFont

**Source:** `panda/src/text/staticTextFont.h` (+ `.I`, `.cxx`)
**Inherits:** [TextFont](TextFont.md)

A font whose glyphs are all pre-generated and loaded from a model produced by
the `egg-mkfont` tool, rather than rasterized on demand. Doesn't require
FreeType — this is the only font type available in a Panda3D build without
`HAVE_FREETYPE`.

## Behavior notes

- **The constructor walks the model tree looking for nodes named as plain
  integers.** `find_characters()` recurses through the `PandaNode` hierarchy;
  any node whose name is entirely digits is treated as the character code for
  a glyph, and `find_character_gsets()` locates that node's two child Geoms:
  the polygon defining visible shape (`ch`), and a single-point `GeomPoints`
  geom (`dot`) whose first vertex encodes the glyph's advance width. A node
  named exactly `"ds"` is special: instead of a character, its `dot` geom
  encodes the font's overall design size — `_line_height` from the Z
  component, `_total_poly_margin` from the X component, and
  `_space_advance = 0.25 * _line_height` (later overridable by an explicit
  space-character glyph).
- **The model is force-converted to `CS_zup_right` before use, unconditionally.**
  If constructed with a `cs` other than `CS_zup_right` (or `CS_default`,
  which resolves to the engine default), the entire subgraph is copied,
  transformed by `LMatrix4::convert_mat(cs, CS_zup_right)`, and flattened —
  because the rest of the text-rendering pipeline assumes glyphs live in
  `CS_zup_right`.
- **Explicit space-character glyph overrides the `"ds"`-derived default.** If
  a glyph exists for character code 32 (space), its `get_advance()` becomes
  `_space_advance`, superseding the `0.25 * _line_height` estimate from the
  design-size node.
- **Font textures get sane defaults only if unset.** Any texture found on the
  model (`find_all_textures()`) has compression forced off unconditionally
  (broken drivers + text readability risk isn't worth the memory savings), but
  quality level / min filter / mag filter are only overridden if they're still
  at their `_default`/`FT_default` values — an explicitly-authored egg file
  can still override these per-texture.
- **`_is_valid` is simply `!_glyphs.empty()`** — a model with zero
  integer-named child nodes produces a "loaded successfully but has no
  glyphs" font, which will render every character as the invalid-glyph
  placeholder.
- **`make_copy()` re-parses the *original* font model from scratch** (`new
  StaticTextFont(_font)`) rather than doing a shallow copy of `_glyphs` — a
  second full tree walk each time.

## API

| Signature | Notes |
|---|---|
| `StaticTextFont(PandaNode *font_def, CoordinateSystem cs = CS_default)` | `font_def` is the root of an egg-mkfont-generated model |
| `virtual PT(TextFont) make_copy() const` | Re-parses `font_def` again (see above) |
| `virtual bool get_glyph(int character, CPT(TextGlyph) &glyph)` | Looks up `_glyphs`; on miss, sets `glyph` to the invalid-glyph placeholder and returns `false` |
| `virtual void write(std::ostream&, int indent_level) const` | Summarizes available characters (groups full alphabets/digit sets, lists odd ones individually) |

## Usage

```cpp
// Typically loaded via FontPool, which picks StaticTextFont automatically
// for .egg/.bam filenames:
PT(TextFont) font = FontPool::load_font("myfont.egg");

// Or explicitly, from an already-loaded model:
PT(PandaNode) model = loader.load_sync("myfont.egg");
PT(StaticTextFont) font = new StaticTextFont(model);
```

## See also

[TextFont.md](TextFont.md) · [DynamicTextFont.md](DynamicTextFont.md) ·
[FontPool.md](FontPool.md) · [README.md](README.md)
