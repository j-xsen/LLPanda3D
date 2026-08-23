# LensNode

**Source:** `panda/src/pgraph/lensNode.h` (+ `.I`, `.cxx`)
**Inherits:** PandaNode **Inherited by:** [Camera](Camera.md), Spotlight (`pgraphnodes`, undocumented)

A node that contains a `Lens` (defined in `mathutil`, undocumented here).
The most important example is [`Camera`](Camera.md), but other nodes
also carry a lens — e.g. a `Spotlight`, which uses its lens to define the
cone of illumination.

## Behavior notes

- A `LensNode` holds a `pvector<LensSlot>` — normally just index 0, but
  multiple lenses can be attached at different indices (see `max-lenses`
  config var), each independently referenceable by index from a
  `DisplayRegion`. Adding a lens via `set_lens(index, lens)` automatically
  marks that slot active.
- `set_lens(Lens*)` shares the pointer — mutating the `Lens` afterward is
  immediately reflected on the node. `copy_lens()` instead calls
  `lens.make_copy()`, decoupling the node from the original.
- `set_lens_active(index, flag)` toggles whether a given lens slot
  participates in rendering; any `DisplayRegion`s tied to an inactive lens
  are implicitly inactive too. Returns `false` if the flag was already set
  (no-op).
- `show_frustum()`/`hide_frustum()` create/remove a child `GeomNode` named
  `"frustum"` built from each active lens's `make_geometry()`, for visual
  debugging — this is why `LensNode::xform()` is overridden but still a
  no-op (the frustum viz would need updating, not yet implemented per the
  `// We need to actually transform the lens here.` comment in the source).
- `is_in_view(pos)` tests a point (in the LensNode's own space) against
  the lens's `make_bounds()` geometric bounding volume — used by
  `Camera`-adjacent code to check camera-visibility of a point without a
  full cull pass.

## API

| Method | Notes |
|---|---|
| `LensNode(name, Lens *lens = nullptr)` | Lens defaults to `nullptr` here (unlike Camera, which defaults to a PerspectiveLens) |
| `set_lens(Lens*)` / `set_lens(index, Lens*)` | Attach a lens by shared pointer |
| `copy_lens(const Lens&)` / `copy_lens(index, const Lens&)` | Attach a *copy* of a lens |
| `get_lens(index = 0)` | Returns nullptr if slot unset |
| `set_lens_active(index, bool)` / `get_lens_active(index)` | Enable/disable a lens slot |
| `activate_lens(index)` / `deactivate_lens(index)` | Shorthand for `set_lens_active` |
| `is_in_view(pos)` / `is_in_view(index, pos)` | Point-in-frustum test |
| `show_frustum()` / `hide_frustum()` | Toggle visible frustum-wireframe child GeomNode |

## Usage

```cpp
PT(LensNode) ln = new LensNode("spot", new PerspectiveLens());
NodePath ln_np = parent.attach_new_node(ln);
ln->show_frustum();  // visualize for debugging
```

## See also

- [Camera](Camera.md) — the primary LensNode subclass
- [GeomNode](GeomNode.md) — used internally for the frustum-visualization child
