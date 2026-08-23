# AccumulatedAttribs

**Source:** `panda/src/pgraph/accumulatedAttribs.h` (+ `.I`, `.cxx`)
**Inherits:** (none — plain value class)

The mutable "working" counterpart to the immutable [RenderState](RenderState.md)/
[TransformState](TransformState.md), used exclusively by
[SceneGraphReducer](SceneGraphReducer.md) while flattening a scene graph. It
walks down the graph pulling specific attrib types (transform, color, color
scale, tex matrix, texture, clip plane, cull face — plus a catch-all
`_other` `RenderState`) off of each node and accumulating them, so they can
later be pushed back down onto (or baked into) the leaves in one combined
step instead of staying scattered across many intermediate nodes.

## Behavior notes

- **`collect()` removes as it accumulates**: `collect(PandaNode*, attrib_types)`
  pulls the node's transform (composing it into `_transform`, then resetting
  the node's own transform to identity) and calls the `RenderState` overload
  to do the same for each tracked attrib slot — each found attrib is
  composed into the matching accumulator field (`_color->compose(node_attrib)`,
  etc.) and then stripped from the node's state via `remove_attrib()`. Which
  attrib types are collected is controlled by an `attrib_types` bitmask of
  `SceneGraphReducer::TransformType` flags (`TT_transform`, `TT_color`,
  `TT_color_scale`, `TT_tex_matrix`, `TT_texture`, `TT_clip_plane`,
  `TT_cull_face`, `TT_other`).
- **Override priority is respected during accumulation**: a node's attrib
  only replaces/composes into the running accumulator if its override value
  is `>=` the accumulator's current override for that slot (or the
  accumulator doesn't have one yet) — so a strongly-overridden attrib
  further down the graph won't be silently clobbered by a weaker one
  encountered earlier.
- **Texture is special-cased**: when accumulating `TT_tex_matrix`, the
  texture attrib is also tracked (composed into `_texture`) but
  deliberately *not* removed from the node — it's only used to know which
  texture-coordinate sets are safe to bake the accumulated tex matrix into,
  not to actually relocate the texture attrib itself.
- **`TT_other` collapses everything else into one state**: rather than
  tracking individual slots, any attrib types not explicitly enumerated
  above get folded wholesale into `_other` via `_other->compose(new_state)`,
  and the node's remaining state becomes empty.
- **`apply_to_node()` is the inverse operation**, called once flattening
  reaches the point where accumulated attribs should land: for each tracked
  type, it composes the accumulator's value on top of the target node's
  existing value (`_color->compose(node_attrib)`, or just the accumulator's
  value if the node had none), calls `get_unique()` to intern the result,
  sets it on the node, and clears that accumulator field back to empty/
  identity so the same `AccumulatedAttribs` instance can be reused for a
  sibling subtree.

## API

| Member/Method | Notes |
|---|---|
| `CPT(TransformState) _transform` | Accumulated transform, identity by default |
| `CPT(RenderAttrib) _color` / `_color_scale` / `_tex_matrix` / `_texture` / `_clip_plane` / `_cull_face` | Per-slot accumulators, `nullptr` until first collected |
| `int _color_override` (+ one per tracked attrib) | Highest override value seen so far for that slot |
| `CPT(RenderState) _other` | Catch-all for everything not individually tracked, empty by default |
| `void collect(PandaNode *node, int attrib_types)` | Pull tracked attribs+transform off a node into the accumulator |
| `CPT(RenderState) collect(const RenderState *state, int attrib_types)` | Same, operating on a state directly; returns the reduced state |
| `void apply_to_node(PandaNode *node, int attrib_types)` | Push accumulated attribs onto a node, then clear the accumulator |
| `void write(ostream&, int attrib_types, int indent_level) const` | Debug printing |

## Usage

Internal to `SceneGraphReducer` — not typically constructed directly by
application code:

```cpp
AccumulatedAttribs attribs;
attribs.collect(some_node, SceneGraphReducer::TT_transform | SceneGraphReducer::TT_color);
// ... recurse into children, then eventually:
attribs.apply_to_node(leaf_node, SceneGraphReducer::TT_transform | SceneGraphReducer::TT_color);
```

## See also

[SceneGraphReducer](SceneGraphReducer.md), [RenderState](RenderState.md),
[TransformState](TransformState.md), [GeomTransformer](GeomTransformer.md).
