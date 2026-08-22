# TextProperties

**Source:** `panda/src/text/textProperties.h` (+ `.I`, `.cxx`)
**Inherited by:** [TextNode](TextNode.md)

A bag of visual/layout attributes for text, where each attribute is
individually "specified or not" (tracked via a `_specified` bitmask) rather
than always having an active value. This is what makes nested
`\x01name\x01...\x02` property pushes (see [README.md](README.md#core-concepts))
compose additively instead of always replacing the whole set — see
`add_properties()` below. `TextNode` inherits this directly so the "top-level"
properties of a block of text are just the `TextNode` itself.

## Behavior notes

- **Every property follows the same has/set/get/clear pattern, backed by one
  bitmask.** `set_X()` stores the value and sets the `F_has_X` bit; `clear_X()`
  resets to the compiled-in default and clears the bit; `has_X()` tests the
  bit; `get_X()` returns the stored value regardless of whether it was
  explicitly set (i.e. `get_X()` is meaningful even when `has_X()` is false —
  it's the default, not garbage). The single exception is `get_font()`, which
  falls back to `get_default_font()` when `!has_font()`, rather than a fixed
  compiled-in default.
- **`add_properties(other)` — used for property-push composition — only
  copies fields `other` has explicitly specified, leaving everything else
  alone.** This is the mechanism that makes `"x\x01up\x01n\x02 + y"` render
  `n` with *only* the `"up"` properties' explicit overrides layered on top of
  whatever was active before the push, not a wholesale replacement.
- **`glyph_scale` and `glyph_shift` are the deliberate exception: they
  compose multiplicatively/additively instead of replacing.** In
  `add_properties()`:
  ```cpp
  set_glyph_shift(other.get_glyph_shift() * get_glyph_scale() + get_glyph_shift());
  set_glyph_scale(other.get_glyph_scale() * get_glyph_scale());
  ```
  So nested pushes of e.g. a "superscript" properties structure stack
  correctly (each level scales/shifts relative to the current level), rather
  than a doubly-nested superscript resetting back to the innermost push's
  absolute scale.
- **Notable non-obvious defaults** (from the default constructor):
  `_align = A_left`, `_text_color = (1,1,1,1)` (opaque white),
  `_shadow_color = (0,0,0,1)` (opaque black), `_draw_order = 1`,
  `_glyph_scale = _text_scale = 1.0`, `_preserve_trailing_whitespace = false`,
  and **`_direction = D_rtl`** — right-to-left is the compiled-in default
  `Direction`, not left-to-right; in practice this field only matters when
  `has_direction()` is true (i.e. `set_direction()` was called) or HarfBuzz
  shaping is active, since HarfBuzz otherwise guesses direction per script,
  so this default rarely surfaces visibly.
- **`get_text_state()`/`get_shadow_state()` cache their computed `RenderState`
  in mutable members, invalidated only by assignment/copy.** Calling either
  builds a `RenderState` from `text_color`/`shadow_color`/`bin`/`draw_order`
  once and caches it (`_text_state`/`_shadow_state`); `operator=()` and the
  copy constructor's copy-then-`.clear()` idiom are the only things that
  invalidate the cache. Mutating a property in place (e.g. `set_text_color()`)
  does **not** itself clear `_text_state` — but since every `set_*` goes
  through `operator=`-based copy paths in normal `TextNode` usage
  (`TextProperties::operator=`), this doesn't surface as a bug in practice;
  it's a detail worth knowing if you hold a raw `TextProperties&` and mutate
  it directly without a fresh copy.
- **The compiled-in default font is loaded exactly once, lazily, on first
  access.** `get_default_font()` checks a static `_loaded_default_font` flag
  and calls `load_default_font()` on first miss; see
  [README.md's "Default font" note](README.md#core-concepts) for what that
  loads. `set_default_font()` marks the flag loaded immediately, so setting
  an explicit default before any text is ever rendered skips the compiled-in
  fallback entirely.
- **`operator==()` only compares fields that are `_specified` on `*this`**,
  using `*this`'s specified-mask, not the union of both sides' masks (and
  requires the masks match exactly first) — two `TextProperties` with
  identical specified-masks and identical values for every bit set in that
  mask compare equal even if... (in practice, since the mask must match
  exactly, this reduces to "both specify the same fields with the same
  values"). There's a duplicated redundant check for `F_has_text_color` in
  the implementation (harmless, just checks the same condition twice).

## API

Every row below follows the `set_X` / `clear_X` / `has_X` / `get_X` pattern
unless noted otherwise.

### Font & shape
| Property | Type | Notes |
|---|---|---|
| `font` | `TextFont*` | `get_font()` falls back to `get_default_font()` when unset |
| `small_caps` | `bool` | Render lowercase as scaled uppercase; default from `text-small-caps` |
| `small_caps_scale` | `PN_stdfloat` | Scale for synthesized small caps; default from `text-small-caps-scale` |
| `slant` | `PN_stdfloat` | Shear factor for synthesized italics |
| `direction` | `Direction` (`D_ltr`/`D_rtl`) | See default-value note above |

### Underscore
| Property | Type | Notes |
|---|---|---|
| `underscore` | `bool` | Draw an underscore line |
| `underscore_height` | `PN_stdfloat` | Relative to baseline; default from `text-default-underscore-height` |

### Layout
| Property | Type | Notes |
|---|---|---|
| `align` | `Alignment` (`A_left/right/center/boxed_left/boxed_right/boxed_center`) | See [TextAssembler.md](TextAssembler.md#alignment) for boxed-mode geometry |
| `indent` | `PN_stdfloat` | First-line indent width |
| `wordwrap` | `PN_stdfloat` | Wrap width; `<= 0` (unset) means no wrapping |
| `preserve_trailing_whitespace` | `bool` | Keep trailing spaces instead of trimming at wrap points |
| `tab_width` | `PN_stdfloat` | Default from `text-tab-width` |

### Color & shadow
| Property | Type | Notes |
|---|---|---|
| `text_color` | `LColor` | Default opaque white |
| `shadow_color` | `LColor` | Default opaque black |
| `shadow` | `LVector2` | Offset; `has_shadow()` gates whether a shadow copy is generated at all |

### Rendering / batching
| Property | Type | Notes |
|---|---|---|
| `bin` | `std::string` | Cull bin name |
| `draw_order` | `int` | Default `1`; text/shadow/frame/card each offset from this — see [TextNode.md](TextNode.md) |
| `glyph_scale` | `PN_stdfloat` | Compounds on nested push — see notes |
| `glyph_shift` | `PN_stdfloat` | Compounds on nested push — see notes |
| `text_scale` | `PN_stdfloat` | Overall scale, replaces (does not compound) on nested push |

### Other
| Signature | Notes |
|---|---|
| `TextProperties()` / `TextProperties(const TextProperties&)` / `operator=` | — |
| `bool operator==/!=(const TextProperties&) const` | See notes on specified-mask semantics |
| `void clear()` | Resets to `TextProperties()` |
| `bool is_any_specified() const` | `_specified != 0` |
| `static void set_default_font(TextFont*)` / `static TextFont *get_default_font()` | Process-wide fallback font |
| `void add_properties(const TextProperties &other)` | Additive merge; see notes |
| `void write(std::ostream&, int indent_level = 0) const` | Dumps only the specified fields |
| `const RenderState *get_text_state() const` / `get_shadow_state() const` | Cached; see notes |

## Usage

```cpp
TextProperties base;
base.set_text_color(1, 1, 1, 1);
base.set_align(TextProperties::A_center);

TextProperties emphasis;
emphasis.set_text_color(1, 0, 0, 1);   // only overrides color

base.add_properties(emphasis);
// base now has A_center alignment (untouched) + red text (overridden)
```

## See also

[TextPropertiesManager.md](TextPropertiesManager.md) · [TextNode.md](TextNode.md)
· [TextAssembler.md](TextAssembler.md) · [README.md](README.md)
