# TextGraphic

**Source:** `panda/src/text/textGraphic.h` (+ `.I`; `.cxx` has no logic, just the include)

A value type describing an arbitrary model to be embedded inline within a
text paragraph, as if it were a printable character. Registered under a name
via [TextPropertiesManager](TextPropertiesManager.md#graphics) and referenced
from a string via the `text-embed-graphic-key` (`\x05`) control character —
see [README.md](README.md#core-concepts).

## Behavior notes

- **The frame is advisory, not enforced.** `get_frame()` (left, right, bottom,
  top) tells [TextAssembler](TextAssembler.md) how much horizontal space to
  reserve and what vertical extent to fold into the line's `line_height`
  calculation, but the actual model is never clipped or scaled to fit it — a
  model larger than its declared frame will visually overlap neighboring
  text.
- **`instance_flag` controls copy-vs-instance semantics, and defaults to
  `false` (copy).** When `false`, [TextAssembler](TextAssembler.md) calls
  `model->copy_subgraph()` for every occurrence — cheaper to render overall
  because scene-graph flattening can combine the copies, but means each
  occurrence is an independent scene-graph subtree. When `true`, the *same*
  node is instanced in directly, wrapped in a `ModelNode` with
  `PT_no_touch` (so it survives flattening/optimization untouched) — needed
  for graphics that must stay identity-linked to the original, such as an
  interactive `PGItem` embedded as text (a button drawn inline, for example),
  where copying would break event routing to the original node.
- **`TextGraphic()`'s default frame is `LVecBase4::zero()`** — a
  default-constructed graphic (as returned by
  `TextPropertiesManager::get_graphic()` for an unregistered name) reserves
  zero space, so it will fully overlap whatever text follows it unless a real
  frame is set.

## API

| Signature | Notes |
|---|---|
| `TextGraphic()` | Empty model, zero frame |
| `TextGraphic(const NodePath &model, const LVecBase4 &frame)` | `frame` is (left, right, bottom, top) |
| `TextGraphic(const NodePath &model, PN_stdfloat left, right, bottom, top)` | Same, unpacked |
| `NodePath get_model() const` / `void set_model(const NodePath&)` | The geometry to embed |
| `LVecBase4 get_frame() const` / `set_frame(...)` | Reserved bounding box; see notes |
| `bool get_instance_flag() const` / `set_instance_flag(bool)` | Copy (default) vs. instance; see notes |

## Usage

```cpp
NodePath icon = window.load_model("icon.egg");
TextGraphic graphic(icon, -0.1f, 1.1f, -0.1f, 1.1f);
TextPropertiesManager::get_global_ptr()->set_graphic("star", graphic);

// In the text string (as a wstring), embed with the key character:
// text_embed_graphic_key + L"star" + text_embed_graphic_key
```

## See also

[TextPropertiesManager.md](TextPropertiesManager.md) · [TextAssembler.md](TextAssembler.md)
· [README.md](README.md)
