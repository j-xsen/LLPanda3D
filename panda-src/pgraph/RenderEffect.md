# RenderEffect

**Source:** `panda/src/pgraph/renderEffect.h` (+ `.I`, `.cxx`)
**Inherits:** TypedWritableReferenceCount
**Inherited by:** BillboardEffect, CompassEffect, DecalEffect, OccluderEffect,
PolylightEffect, ScissorEffect, ShowBoundsEffect, TexProjectorEffect

Abstract base for special per-node behaviors — billboarding, decaling,
compass-facing, portal/occluder marking — that must be applied as soon as
they're encountered on a node, in contrast to [RenderAttrib](RenderAttrib.md)
(appearance properties like color/texture that simply propagate down to
leaves without acting on the node they're found on). Held on `PandaNode` as
a `CPT(RenderEffects)` set, not individually. Never constructed directly;
each subclass provides its own `make()`.

## Behavior notes

- **Simpler composition than RenderAttrib**: `RenderEffects` (plural) is
  just a flat set keyed by `TypeHandle` — there's no `compose()`/override-
  priority mechanism like `RenderState` has, since effects apply locally to
  the node they're on rather than propagating and combining across the
  graph.
- Default virtual answers (all overridable per subclass):
  `safe_to_transform()` → `true`, `safe_to_combine()` → `true`,
  `has_cull_callback()` → `false`, `has_adjust_transform()` → `false`. An
  effect that isn't safe to flatten/combine (e.g. `CompassEffect`, which
  depends on a specific node identity) overrides `safe_to_transform`/
  `safe_to_combine` to `false` so `SceneGraphReducer`/`NodePath::flatten_*()`
  leave it alone.
- `xform(mat)` lets an effect respond to a transform being applied to its
  node (default: return `this` unchanged) — relevant to effects with
  spatial data of their own.
- `prepare_flatten_transform(net_transform)` / `adjust_transform(...)` are
  the hooks `PandaNode::apply_attribs_from()`-style flattening and the cull
  traversal use to let an effect intercept or rewrite the transform that
  would otherwise apply — e.g. `BillboardEffect` uses `adjust_transform` to
  substitute a facing-the-camera rotation for the node's authored one.
- `interning`: like `RenderAttrib`, `return_new()` shares a common pointer
  for equivalent effects via a global `compare_to_impl()`-keyed set — this
  is unconditional (no `uniquify_*` config gate the way attribs/states have).

## API

| Method | Notes |
|---|---|
| `virtual bool safe_to_transform() const` | Can this effect survive an arbitrary matrix transform applied to its node? |
| `virtual bool safe_to_combine() const` | Can this effect survive being combined with a sibling during flattening? |
| `virtual CPT(RenderEffect) xform(const LMatrix4 &mat) const` | Respond to a transform being baked into the node |
| `virtual CPT(TransformState) prepare_flatten_transform(net_transform) const` | Flattening hook |
| `virtual bool has_cull_callback() const` / `cull_callback(trav, data, node_transform, node_state) const` | Cull-time hook (e.g. billboard facing computation) |
| `virtual bool has_adjust_transform() const` / `adjust_transform(net_transform, node_transform, node) const` | Transform-substitution hook |
| `int compare_to(other) const` | Ordering for interning |
| `static int get_num_effects()` / `list_effects(ostream&)` | Global interning-table introspection |

## Usage

```cpp
node_path.set_effect(BillboardEffect::make_axis());
```

## See also

[RenderEffects](RenderEffects.md), [RenderAttrib](RenderAttrib.md),
[RenderState](RenderState.md), [PandaNode](PandaNode.md),
[SceneGraphReducer](SceneGraphReducer.md).
