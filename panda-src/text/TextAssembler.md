# TextAssembler

**Source:** `panda/src/text/textAssembler.h` (+ `.I`, `.cxx`)

The layout engine: turns a `wstring` plus a base [TextProperties](TextProperties.md)
into placed glyphs and, ultimately, `Geom`s under a `PandaNode`. Not normally
used directly — [TextNode](TextNode.md) owns one internally (constructed
fresh in `do_generate()`) — but it's a standalone class user code can use for
low-level text operations (e.g. querying wordwrapped layout without
generating geometry).

## Pipeline

```
set_wtext(wtext)
  └─ scan_wtext()      — parse \1/\2/\5 control chars into TextString
                          (one TextCharacter per char, each carrying a
                          ComputedProperties pointer for the active nested state)
  └─ wordwrap_text()   — break TextString into TextBlock (a vector of TextRow),
                          applying wordwrap width / hyphenation / never-break rules

assemble_text()
  └─ assemble_paragraph()  — for each TextRow: assemble_row(), then position
                              the row (alignment + vertical stacking), updating
                              _ul/_lr
      └─ assemble_row()        — per character: look up glyph(s)
                                  (get_character_glyphs(), cheesy accents via
                                  tack_on_accent()), or shape via HarfBuzz
                                  (shape_buffer()) if enabled; produces
                                  PlacedGlyphs (GlyphPlacement list)
  └─ (in assemble_text) sort placements into quads (generate_quads()) or
     merged Geoms (GeomCollector) or individual per-glyph Geoms, depending on
     set_dynamic_merge()
```

## Behavior notes

### Control-character parsing (`scan_wtext`)

- **Recursive-descent over the string, one call frame per nesting depth.**
  A `\x01name\x01` push recursively calls `scan_wtext()` with a new
  `ComputedProperties` (built from `TextPropertiesManager::get_properties(name)`
  merged onto the current one via `add_properties()` — see
  [TextProperties.md](TextProperties.md#behavior-notes)); the recursive call
  returns when it hits the matching `\x02` pop, or the end of string. An
  unmatched pop at the top level is logged as a warning and parsing resumes;
  an unmatched (unclosed) push is only debug-logged, since trailing pushes
  without a pop are considered acceptable sloppiness.
- **Surrogate pairs are handled explicitly when `wchar_t` is 16-bit**
  (`WCHAR_MAX < 0x10FFFF`, i.e. Windows) — a high surrogate not followed by a
  valid low surrogate logs a warning and is dropped, rather than being passed
  through as a raw code unit.
- **`ComputedProperties` forms a chain, not a flat list**, so that
  `append_delta()` (used by `get_wtext()`/`get_wordwrapped_wtext()` to
  reconstruct a wtext with the *minimum* necessary push/pop sequence when
  round-tripping already-parsed text) can diff two arbitrary points in the
  nesting by walking up to a common ancestor depth first.

### Wordwrapping (`wordwrap_text`)

- **Hyphenation is soft-hyphen-driven, not dictionary-based.** A hyphenation
  point only exists where the text explicitly contains a
  `text-soft-hyphen-key` character; there is no automatic word-breaking
  algorithm. When a line overflows, the algorithm prefers (in order): the
  right-most soft hyphen *if* the right-most breakable space isn't already
  close enough to the wrap width (`text-hyphen-ratio` — if a space falls
  within e.g. the last 30% of the line, don't bother hyphenating), then the
  right-most breakable space, then a forced break pulled back from
  `text-never-break-before` punctuation (capped at `text-max-never-break`
  characters of pull-back, so a run of many forbidden characters doesn't
  degrade to breaking arbitrarily far back).
- **`wordwrap_width` is re-read from `TextProperties` per-character during
  the scan, not fixed for the whole row** — because each character carries
  its own `ComputedProperties` (from nested property pushes), a wordwrap
  width set inside a pushed properties structure only applies from that point
  in the row onward, and reverts (to `-1`, meaning "unset") the moment a
  character without `has_wordwrap()` is reached. This is a fairly obscure
  edge case in practice (wordwrap is normally set once, at the `TextNode`
  level).
- **A single character wider than the entire wordwrap width still gets
  emitted** — `wordwrap_text()`'s fallback for "no characters fit at all"
  lets the offending character in anyway rather than producing an empty line
  or infinite-looping, but only when there was no leading whitespace
  contributing to `initial_width`; leading-whitespace-caused overflow is
  handled by a different branch that just accepts the whitespace.
- **`get_multiline_mode() == false` disables the "start a new row" behavior**
  entirely (`needs_newline` never becomes `true`), but *embedded* `\n`
  characters still start new `TextRow`s regardless — multiline mode only
  suppresses wordwrap-induced row breaks, not explicit newlines.

### Glyph placement & alignment {#alignment}

- **Row height is the max line height across every font/scale used in the
  row**, including graphics (`frame[3] - frame[2]`) and, if the row ends on
  an embedded newline, the properties active at that newline even if the row
  itself was otherwise empty (so a lone `\n` in a large font still takes up
  that font's line height).
- **The "boxed" alignment modes (`A_boxed_left/right/center`) size against
  the wordwrap width, not the row's actual content width** — e.g.
  `A_boxed_left` sets the paragraph's right edge (`_lr[0]`) to
  `max(row_width, wordwrap)`, so a short line in boxed alignment still
  reserves the full wordwrap-width box, unlike plain `A_left` which only
  reserves as much as the longest actual row.
- **Kerning is applied only in the non-HarfBuzz path**, gated by
  `text-kerning`, tracking `prev_char` across characters (reset to `-1` after
  a space, tab, graphic, or HarfBuzz-shaped run) and calling
  `font->get_kerning(prev_char, character)`.
- **Underscore geometry batches a whole contiguous run at the row's active
  color/height as one line segment** (`draw_underscore()`) — it only emits
  geometry when the underscore setting, text color, or underscore height
  *changes*, not per-character, so contiguous underscored text of the same
  color produces one line, not one per glyph.

### Cheesy accents and ligatures

- **`get_character_glyphs()` only attempts synthesis after a direct glyph
  lookup fails.** It first optionally remaps to uppercase for small caps,
  then tries the font directly; only on failure does it consult
  `UnicodeLatinMap` for an ASCII base-letter equivalent (with special-cased
  dotless-i/dotless-j handling for Turkish-style accented forms) and,
  separately, a possible second glyph for a synthesized ligature
  (`AF_ligature`, whose combined advance is scaled down by
  `ligature_advance_scale = 0.6`) or small-cap scale (`AF_smallcap`).
- **Accent placement is table-driven via `CheesyPosition`/`CheesyTransform`
  enums** (above/below/top/bottom/within, combined with
  none/mirror/rotate-90/180/270/squash variants/small/tiny scalings) —
  `tack_on_accent()` positions a plain combining-mark glyph relative to the
  base glyph's tight bounding box and centroid. This is a fixed, hand-tuned
  lookup table per `AccentType`, not a general font-metrics-driven
  positioning system, and produces a visually-approximate result rather than
  true font hinting.
- **`AF_turned` (e.g. n-tilde-like inverted forms) rotates the base glyph
  around its own centroid** by negating `_scale` and repositioning by
  `2 * centroid`, and assumes (per a comment in the source) that no character
  needing this flag also needs an accent mark tacked on above/below.

### HarfBuzz shaping {#harfbuzz-shaping}

- **Only engages for `DynamicTextFont`, and only when `text-use-harfbuzz` is
  set.** `assemble_row()` checks `font->is_of_type(DynamicTextFont::get_class_type())`
  before creating an `hb_buffer_t`; `StaticTextFont` text always uses the
  plain per-character path regardless of the config variable.
  `HAVE_HARFBUZZ` must also be compiled in, or all of this compiles out.
- **Buffering is chunked by `ComputedProperties` identity, not by row.**
  Every time the active `ComputedProperties` pointer changes (a push/pop
  boundary) or an embedded graphic is hit, any accumulated buffer is shaped
  immediately (`shape_buffer()`) and reset — so a run of pushed/popped
  properties within one row causes multiple separate HarfBuzz shaping calls,
  each only seeing its own contiguous run of same-properties text (HarfBuzz
  needs a contiguous run to shape ligatures/context correctly, so crossing a
  properties boundary mid-run would produce wrong shaping anyway).
- **Small-caps characters are flagged via a high bit smuggled into the
  HarfBuzz cluster value** (`ClusterFlags::CF_small_caps = 0x200000`) rather
  than a separate side channel — `character | CF_small_caps` is passed as the
  cluster id, and `shape_buffer()` masks it back out (`cluster & 0x1fffff`)
  to recover the real character while checking the flag bit to apply
  `small_caps_scale` to that glyph's advance/scale.

### Quads vs. arbitrary Geoms {#quads-vs-arbitrary-geoms}

See also [README.md's note on this](README.md#core-concepts).

- **`generate_quads()` hand-writes vertex/index buffers via raw pointers**,
  with separate 32-bit-float and 64-bit-float (`PN_stdfloat` build
  configuration) code paths, specifically because `GeomVertexWriter` was
  measured as the bottleneck for large amounts of text — this is one of the
  few genuinely perf-critical hot loops in the module, called once per unique
  `RenderState` with all its quads batched into one `GeomTriangles`.
  Index width (16-bit vs. 32-bit) is chosen based on whether the batch
  exceeds 65535 vertices.
- **`GeomCollector` deduplicates vertices per source primitive when merging
  non-quad Geoms**, using a per-append `VertexIndexMap` so a single glyph's
  own internal shared vertices (e.g. a triangle fan) aren't needlessly
  duplicated in the merged buffer — but the map is scoped to one
  `assign_append_to()` call (one glyph), so vertices are never deduplicated
  *across* different glyphs even if they happen to coincide.
- **When `dynamic_merge` is off, decorations lose their batching entirely** —
  `GlyphPlacement::assign_to()` creates one independent `Geom` per glyph, with
  the expectation that `TextNode`'s subsequent `flatten_strong()`/etc. pass
  will recombine them at the scene-graph level instead.

## API

### Setup
| Signature | Notes |
|---|---|
| `TextAssembler(TextEncoder *encoder)` | `encoder` is used for `\5`-graphic name decoding |
| `void clear()` | Resets to empty text |
| `void set_usage_hint(Geom::UsageHint)` / `get_usage_hint()` | Passed to generated `GeomVertexData` |
| `void set_max_rows(int)` / `get_max_rows()` | 0 = unlimited |
| `void set_dynamic_merge(bool)` / `get_dynamic_merge()` | See "Quads vs. arbitrary Geoms" above |
| `void set_multiline_mode(bool)` / `get_multiline_mode()` | Default `true`; see wordwrap notes |
| `void set_properties(const TextProperties&)` / `get_properties()` | The base/default properties for unspecified text |

### Text input
| Signature | Notes |
|---|---|
| `bool set_wtext(const std::wstring&)` | Parses + wordwraps; returns `false` if truncated by `max_rows` |
| `bool set_wsubstr(const std::wstring&, int start, int count)` | Splice-replace a range of already-parsed characters |
| `std::wstring get_plain_wtext() const` | Control chars stripped, embedded graphics become `0` |
| `std::wstring get_wordwrapped_plain_wtext() const` | Same, post-wordwrap (embedded newlines added) |
| `std::wstring get_wtext() const` / `get_wordwrapped_wtext() const` | Reconstructed with minimal push/pop sequences (`append_delta()`) |

### Indexing
| Signature | Notes |
|---|---|
| `bool calc_r_c(int &r, int &c, int n) const` | Row/column for the nth character; `false` for non-positioned chars (soft hyphens, newlines) |
| `int calc_r(int n)` / `calc_c(int n)` | Convenience wrappers, `-1` on failure |
| `int calc_index(int r, int c) const` | Inverse of `calc_r_c` |
| `int get_num_characters() const` / `char32_t get_character(int n) const` / `get_graphic(int n)` / `get_properties(int n)` / `get_width(int n)` | Pre-wordwrap indexing |
| `int get_num_rows() const` / `get_num_cols(int r) const` / per-row `get_character/get_graphic/get_properties/get_width(r, c)` | Post-wordwrap indexing |
| `PN_stdfloat get_xpos(int r, int c) const` / `get_ypos(int r, int c) const` | Position of a specific character |

### Assembly
| Signature | Notes |
|---|---|
| `PT(PandaNode) assemble_text()` | Runs the full placement pipeline; returns a `PandaNode` with `shadow`/`text` children |
| `const LVector2 &get_ul() const` / `get_lr() const` | Bounding box of the assembled text, valid after `assemble_text()` |

### Static width/character queries
| Signature | Notes |
|---|---|
| `static PN_stdfloat calc_width(wchar_t/char32_t, const TextProperties&)` | Handles space specially; accounts for cheesy-accent/ligature width |
| `static PN_stdfloat calc_width(const TextGraphic*, const TextProperties&)` | Frame width × glyph/text scale |
| `static bool has_exact_character/has_character/is_whitespace(wchar_t, const TextProperties&)` | See [TextFont.md](TextFont.md) for the underlying distinction |

## See also

[TextNode.md](TextNode.md) · [TextProperties.md](TextProperties.md) ·
[TextPropertiesManager.md](TextPropertiesManager.md) · [TextGraphic.md](TextGraphic.md)
· [TextGlyph.md](TextGlyph.md) · [GeomTextGlyph.md](GeomTextGlyph.md) ·
[DynamicTextFont.md](DynamicTextFont.md) · [README.md](README.md)
