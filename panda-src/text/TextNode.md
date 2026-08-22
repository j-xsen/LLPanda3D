# TextNode

**Source:** `panda/src/text/textNode.h` (+ `.I`, `.cxx`)
**Inherits:** `PandaNode`, `TextEncoder` (`panda/src/putil`), [TextProperties](TextProperties.md)

The primary user-facing interface to this module: a `PandaNode` that holds a
text string (via inherited `TextEncoder`) and visual properties (via
inherited `TextProperties`), and lazily converts them into renderable
geometry on demand via [TextAssembler](TextAssembler.md). Can be parented
directly into the scene graph and rendered like any node, or used purely as a
generator via `generate()`.

## Behavior notes

- **Rebuilding is lazy and dirty-flag-driven, guarded by a per-instance
  `Mutex`.** Setters like `set_text_color()`, `set_wordwrap()`, `set_font()`
  (all overridden here vs. the `TextProperties` base) just flip
  `F_needs_rebuild` and/or `F_needs_measure` and return — no assembly work
  happens until something actually needs the result. Two internal helpers
  gate the real work: `check_rebuild()` calls `do_rebuild()` if
  `F_needs_rebuild` is set; `check_measure()` calls `do_measure()` if
  `F_needs_measure` is set. **`do_measure()` doesn't actually do a cheaper
  measure-only pass — it just calls `do_rebuild()`.** So even a
  measurement-only query (`get_left()`, `get_num_rows()`, etc.) can trigger a
  full `TextAssembler` run if the text is dirty.
- **`invalidate_no_measure()` vs. `invalidate_with_measure()`**: the former
  (used for things like color changes that can't affect layout) sets only
  `F_needs_rebuild`; the latter (text changes, font changes, scale changes)
  additionally sets `F_needs_measure` and calls `mark_internal_bounds_stale()`.
  Given the `do_measure() == do_rebuild()` behavior above, this distinction
  currently doesn't change *what* work happens on the next access, only
  whether the bounding volume is also invalidated.
- **`cull_callback()` is where the lazy rebuild actually gets triggered
  during normal rendering** — it calls `do_get_internal_geom()` (which calls
  `check_rebuild()` under the lock) every time the node is culled, then
  manually traverses into the resulting geometry via
  `trav->traverse(next_data)`. This means a `TextNode` sitting in the scene
  graph but never actually visible (culled by the view frustum before
  `cull_callback` runs) never assembles its text at all.
- **`do_generate()` builds a small sub-hierarchy, then flattens it back
  down**, according to `_flatten_flags` (`FF_none`/`light`/`medium`/`strong`,
  defaulting from `text-flatten`) — the row/character transform hierarchy
  `TextAssembler` conceptually implies is actually flattened away by
  `NodePath::flatten_strong()`/etc. before the frame/card decorations are
  added, so what ends up in the scene graph is much shallower than the
  logical row-by-row structure.
- **Frame and card are added as separate steps, in a specific order, and
  cards can "adopt" the text as a child.** `do_rebuild()`'s order is: build
  text → flatten → build card (if any) → **if a card exists, `card_root`
  becomes the new parent of `root`** (so the card renders behind/around the
  text) → build frame (added as a sibling child of the current root, after
  the card reparenting). If `F_card_decal` is set, a `DecalEffect` is applied
  to the card so the text polygons don't z-fight with it — necessary for
  rendering text directly in 3-D space without a dedicated bin.
- **Draw order offsets are baked in, not configurable per-decoration.** Given
  `TextProperties::get_draw_order()` as `n`: the shadow copy of the text
  renders in bin-order `n+1` via `get_shadow_state()`, the frame in `n+1`,
  the main text in `n+2`, and the card underneath at `n` — see
  `TextProperties::get_text_state()`/`get_shadow_state()` for where `+1`/`+2`
  actually get added to the bin's draw order.
- **`apply_attribs_to_vertices()` special-cases flat color/color-scale
  attribs rather than pushing them down to geometry.** A flat `ColorAttrib`
  updates `text_color`, `shadow_color`, `_frame_color`, and `_card_color` all
  at once (`invalidate_no_measure()`, no full remeasure needed); a
  `ColorScaleAttrib` component-wise multiplies the same four colors. Anything
  already generated and cached (`_internal_geom`, when not already dirty) is
  additionally pushed through a `SceneGraphReducer` to apply attributes
  directly to existing vertices — meaning a flattening pass touching an
  already-built `TextNode` mutates its stored colors in place rather than
  just wrapping it in a state change.
- **`get_unsafe_to_apply_attribs()` refuses texture-matrix and "other"
  attribute categories.** `TextNode` has no mechanism to bake a tex-matrix
  transform into its glyph UVs, so the scene graph flattener is told to leave
  those as a state change at this node instead of attempting to push them
  into vertices.
- **`compute_internal_bounds()` can compute a bounding volume from only a
  measure, without ever generating geometry** — it calls `check_measure()`
  (which, per the note above, currently does a full rebuild anyway) and
  builds a `BoundingSphere` around the 8 corners of the `_ul3d`/`_lr3d` box,
  entirely independent of whether `_internal_geom` exists.
- **The `make_card_with_border()` path builds three tri-strips for a 9-slice
  style border** (corners + edges + center), driven by `card_border_size` and
  `card_border_uv_portion` — only used when `F_has_card_border` is set via
  `set_card_border()`; the plain `make_card()` path is a single quad.

## API

### Text query (delegates to `TextAssembler` statics)
| Signature | Notes |
|---|---|
| `PN_stdfloat calc_width(wchar_t) const` / `calc_width(const std::wstring&) const` | Width in the current font, ignoring kerning |
| `bool has_exact_character(wchar_t) const` | In the font exactly, no synthesis |
| `bool has_character(wchar_t) const` | In the font, or synthesizable |
| `bool is_whitespace(wchar_t) const` | — |
| `std::string get_wordwrapped_text() const` / `std::wstring get_wordwrapped_wtext() const` | Text as it was actually wrapped, valid after a rebuild |

### Rebuild control
| Signature | Notes |
|---|---|
| `PT(PandaNode) generate()` | Returns a standalone node with the current text's geometry; call repeatedly for independent copies |
| `void update()` | Forces `check_rebuild()` now instead of at next cull |
| `void force_update()` | Rebuilds unconditionally, even if not marked dirty |
| `PT(PandaNode) get_internal_geom() const` | Debug-only; logs a warning when called |

### Sizing / overflow
| Signature | Notes |
|---|---|
| `void set_max_rows(int)` / `clear_max_rows()` / `has_max_rows()` / `get_max_rows()` | Caps rows; excess is truncated |
| `bool has_overflow() const` | True if the last assembly truncated text due to `max_rows` |
| `PN_stdfloat get_left/right/bottom/top/height/width() const` | From the last measure/build |
| `LPoint3 get_upper_left_3d() const` / `get_lower_right_3d() const` | `_ul3d`/`_lr3d`, transformed |
| `int get_num_rows() const` | — |

### Frame (outline)
| Signature | Notes |
|---|---|
| `void set_frame_as_margin(l, r, b, t)` / `set_frame_actual(l, r, b, t)` / `clear_frame()` | Margin-relative vs. absolute coordinates |
| `bool has_frame() const` / `is_frame_as_margin() const` | — |
| `LVecBase4 get_frame_as_set() const` / `get_frame_actual() const` | Raw set values vs. resolved (measured) actual values |
| `void set_frame_color/get_frame_color(const LColor&)` | — |
| `void set_frame_line_width(PN_stdfloat)` / `get_frame_line_width()` | — |
| `void set_frame_corners(bool)` / `get_frame_corners()` | Draw point sprites at corners to soften thick lines |

### Card (background quad)
| Signature | Notes |
|---|---|
| `void set_card_as_margin(l, r, b, t)` / `set_card_actual(l, r, b, t)` / `clear_card()` | Same margin-vs-absolute pattern as frame |
| `bool has_card() const` / `is_card_as_margin() const` | — |
| `LVecBase4 get_card_as_set() const` / `get_card_actual() const` / `get_card_transformed() const` | Set / measured / world-transformed |
| `void set_card_color/get_card_color(const LColor&)` | — |
| `void set_card_texture(Texture*)` / `clear_card_texture()` / `has_card_texture()` / `get_card_texture()` | — |
| `void set_card_border(PN_stdfloat size, PN_stdfloat uv_portion)` / `clear_card_border()` / `has_card_border()` | 9-slice border mode; see notes |
| `void set_card_decal(bool)` / `get_card_decal()` | See `DecalEffect` note above |

### Transform / general
| Signature | Notes |
|---|---|
| `void set_transform(const LMatrix4&)` / `get_transform()` | Extra transform on the whole paragraph |
| `void set_coordinate_system(CoordinateSystem)` / `get_coordinate_system()` | Text is built in `CS_zup_right`, then converted |
| `void set_usage_hint(Geom::UsageHint)` / `get_usage_hint()` | Passed to generated `GeomVertexData` |
| `void set_flatten_flags(int)` / `get_flatten_flags()` | `FF_none/light/medium/strong` \| `FF_dynamic_merge` |

### TextProperties setters (overridden to also invalidate)
All of `set_font/set_small_caps/set_small_caps_scale/set_slant/set_align/
set_indent/set_wordwrap/set_text_color/set_shadow_color/set_shadow/set_bin/
set_draw_order/set_tab_width/set_glyph_scale/set_glyph_shift` (+ their
`clear_*` counterparts) are re-declared here purely to call
`invalidate_no_measure()`/`invalidate_with_measure()` after delegating to the
`TextProperties` base implementation.

## Usage

```cpp
PT(TextNode) text = new TextNode("my_text");
text->set_text("Hello, World!");
text->set_align(TextNode::A_center);
text->set_text_color(1, 1, 1, 1);
text->set_shadow(0.05, -0.05);
NodePath text_np = aspect2d.attach_new_node(text);

// Or, as a pure generator (not parented):
PT(TextNode) gen = new TextNode("gen");
gen->set_text("Static label");
NodePath label = some_parent.attach_new_node(gen->generate());
```

## See also

[TextProperties.md](TextProperties.md) · [TextAssembler.md](TextAssembler.md) ·
[TextFont.md](TextFont.md) · [README.md](README.md)
