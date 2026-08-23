# SceneGraphReducer

**Source:** `panda/src/pgraph/sceneGraphReducer.h` / `.I` / `.cxx`

Implements scene-graph "flattening" — collapsing unneeded nodes and
folding state/transform changes into geometry to reduce node count and
state-change overhead. `NodePath::flatten_light()`/`flatten_medium()`/
`flatten_strong()` construct a `SceneGraphReducer` internally with
progressively more aggressive `combine_siblings_bits` and delegate to
`flatten()`; designed to be subclassed for custom flattening policy, though
the defaults normally suffice.

## Behavior notes

- **`flatten()`'s `combine_siblings_bits`** (an `CombineSiblings` mask —
  `CS_geom_node`, `CS_within_radius`, `CS_other`, `CS_recurse`) controls how
  aggressively sibling nodes get merged together, not just parent-child
  collapsing: `CS_geom_node` merges sibling `GeomNode`s,
  `CS_within_radius` (paired with `set_combine_radius()`) only merges
  siblings whose bounding volumes are within a given radius of each other
  (avoiding merging geometry that's far apart in the scene, which would
  bloat bounding volumes), `CS_other` merges non-GeomNode siblings, and
  `CS_recurse` applies sibling-combining recursively rather than once.
- **`apply_attribs()` pushes accumulated state down into geometry** via an
  [AccumulatedAttribs](AccumulatedAttribs.md) accumulator and a
  [GeomTransformer](GeomTransformer.md) — `attrib_types` (a `AttribTypes`
  mask: `TT_transform`, `TT_color`, `TT_color_scale`, `TT_tex_matrix`,
  `TT_clip_plane`, `TT_cull_face`, `TT_apply_texture_color`, `TT_other`)
  selects which categories of state get baked into vertices vs. left as
  node-level `RenderAttrib`s. The default mask excludes
  `TT_clip_plane | TT_cull_face | TT_apply_texture_color` — those three are
  opt-in since baking them changes rendering semantics in ways not always
  desired.
- `collect_vertex_data()`'s bits are a separate `CollectVertexData` enum
  (`CVD_name`, `CVD_model`, `CVD_transform`, `CVD_avoid_dynamic`,
  `CVD_one_node_only`, `CVD_format`, `CVD_usage_hint`, `CVD_animation_type`)
  controlling which `GeomVertexData`s are allowed to merge with which —
  e.g. `CVD_model` stops merging at `ModelNode` boundaries so a loaded
  model's internal geometry doesn't get fused with an unrelated sibling
  model's.
- `premunge()` runs a GSG-specific vertex-munging pass (see `StateMunger`)
  over the whole subtree — this is what `premunge-data` triggers on freshly
  loaded models in [Loader](Loader.md).
- `check_live_flatten()` is the gate behind the `allow-live-flatten` config
  var — refuses to flatten a subtree that's currently attached under
  `render` unless that variable is set, since flattening a live subtree can
  race with rendering.
- Each major operation (`flatten`, `apply_attribs`, `remove_column`,
  `make_compatible_state`, `collect_vertex_data`, `make_nonindexed`,
  `unify`, `remove_unused_vertices`, `premunge`) has its own static
  `PStatCollector` for profiling.

## API (grouped by purpose)

| Method | Purpose |
|---|---|
| `explicit SceneGraphReducer(GraphicsStateGuardianBase *gsg = nullptr)` | GSG needed for GSG-dependent ops like `premunge()`/`make_nonindexed()` |
| `int flatten(PandaNode *root, int combine_siblings_bits)` | Main entry point |
| `void apply_attribs(PandaNode*, int attrib_types = ~(TT_clip_plane\|TT_cull_face\|TT_apply_texture_color))` | Bake state into geometry |
| `int remove_column(PandaNode*, const InternalName*)` | Strip a vertex column (e.g. unused texcoords) throughout |
| `int make_compatible_state(PandaNode*)` | Normalize state so more merging is possible |
| `int make_compatible_format(PandaNode*, collect_bits = ~0)` | Normalize vertex formats |
| `void decompose(PandaNode*)` | Splits composite primitives (tristrips/fans) into simple triangles |
| `int collect_vertex_data(PandaNode*, collect_bits = ~0)` | Merge vertex buffers (`CollectVertexData` bits) |
| `int make_nonindexed(PandaNode*, nonindexed_bits = ~0)` | Convert indexed geometry to nonindexed (`MakeNonindexed` bits) |
| `void unify(PandaNode*, bool preserve_order)` | Combine primitives within a Geom into fewer draw calls |
| `void remove_unused_vertices(PandaNode*)` | Drop vertices no primitive references |
| `void premunge(PandaNode*, const RenderState *initial_state)` | GSG vertex-format munging pass |
| `bool check_live_flatten(PandaNode*)` | Guards flattening a live/attached subtree |
| `void set_combine_radius(PN_stdfloat)` / `get_combine_radius()` | Used with `CS_within_radius` |
| `void set_gsg(GraphicsStateGuardianBase*)` / `clear_gsg()` / `get_gsg()` | |

## See also

- [GeomTransformer](GeomTransformer.md), [AccumulatedAttribs](AccumulatedAttribs.md), [ModelFlattenRequest](ModelFlattenRequest.md)
