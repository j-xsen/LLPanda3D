# GeomTextGlyph

**Source:** `panda/src/text/geomTextGlyph.h` (+ `.I`, `.cxx`)
**Inherits:** `Geom`

A `Geom` subclass whose only purpose is to carry a list of the
[TextGlyph](TextGlyph.md)s that contributed vertices to it, so glyph reference
counts survive `Geom` copying/merging. See
[README.md's GeomTextGlyph note](README.md#core-concepts) for why this
matters for garbage collection.

## Behavior notes

- **Every `Geom` a `TextGlyph::get_geom()` hands out is actually one of
  these**, constructed with the originating glyph already in `_glyphs`. Plain
  `Geom`s (not `GeomTextGlyph`) are used internally as the *cached template*
  a quad glyph is copied from (`TextGlyph::make_quad_geom()`), specifically to
  avoid a self-referential cycle — the copy made for external use is what gets
  wrapped as a `GeomTextGlyph`.
- **`copy_primitives_from()` and `count_geom()` both propagate `_glyphs`, but
  for different reasons.** `copy_primitives_from()` is the normal `Geom` API
  used by `TextAssembler::GeomCollector`-style merging and genuinely appends
  primitives; `count_geom()` does *not* copy any geometry — it exists purely
  so a ref-count bookkeeping pass can register "this merged Geom now also
  counts as using these other glyphs" without an actual primitive copy.
- **Registered with the `BamReader` factory** (`register_with_read_factory()`,
  called from `init_libtext()`) so `GeomTextGlyph` objects can round-trip
  through `.bam` files — though the `_glyphs` list itself is not persisted
  (there's no `write_datagram`/`fillin` override here beyond what `Geom`
  provides), so a glyph reloaded from `.bam` starts with an empty `_glyphs`.
- **`friend class TextAssembler`** — `TextAssembler::generate_quads()` reaches
  in and directly swaps into `_glyphs` when building the batched-quad `Geom`,
  bypassing `add_glyph()`/`count_geom()` for performance.

## API

| Signature | Notes |
|---|---|
| `GeomTextGlyph(const TextGlyph *glyph, const GeomVertexData *data)` | Starts with one glyph already tracked |
| `GeomTextGlyph(const GeomVertexData *data)` | Starts with zero glyphs tracked |
| `GeomTextGlyph(const Geom &copy, const TextGlyph *glyph)` | Wraps an existing plain `Geom`, adding one glyph |
| `void add_glyph(const TextGlyph *glyph)` | Appends one glyph to the tracked list |
| `virtual bool copy_primitives_from(const Geom *other)` | Normal `Geom` primitive copy + glyph-list append; requires `other` also be a `GeomTextGlyph` |
| `void count_geom(const Geom *other)` | Glyph-list append only, no primitive copy; no-op if `other` isn't a `GeomTextGlyph` |
| `virtual Geom *make_copy() const` | Shallow copy, glyph list included |
| `static TypeHandle get_class_type()` | — |

## See also

[TextGlyph.md](TextGlyph.md) · [TextAssembler.md](TextAssembler.md) ·
[DynamicTextFont.md](DynamicTextFont.md#garbage-collection) · [README.md](README.md)
