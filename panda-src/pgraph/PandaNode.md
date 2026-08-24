# PandaNode

**Source:** `panda/src/pgraph/pandaNode.h` (+ `.I`, `.cxx`)
**Inherits:** TypedWritableReferenceCount, Namable, LinkedListNode
**Inherited by:** LensNode, Fog, GeomNode, ModelNode, OccluderNode, PlaneNode, PolylightNode, PortalNode, and every other scene-graph node type

The base class of every scene graph node — a generic node with no special
properties by default, and the type application code almost never touches
directly (see [NodePath](NodePath.md) for why). Application code that
*implements* a custom node type subclasses this and overrides its virtuals
(`cull_callback()`, `is_renderable()`, `safe_to_flatten()`,
`compute_internal_bounds()`, etc.).

## Behavior notes

- **Multiple parents are legal.** Unlike most scene-graph libraries, a
  `PandaNode` keeps a full `Up` list (parents) as well as a `Down` list
  (children) — the same node can be instanced under several different
  parents at once, which is how model instancing works. This is also why a
  `PandaNode*` alone cannot disambiguate which occurrence in the graph is
  meant — that is `NodePath`'s job. Child pointers are ref-counted
  (`PT(PandaNode)`); parent pointers are **not** (raw `PandaNode*`), so
  parent/child references don't form a reference cycle — a node's lifetime
  is governed by its children/instance references, not by its parents.
- **`add_child()` replaces, and rejects cycles.** Re-adding the same child
  to a node first silently removes its previous instance under that parent;
  `add_child()` calls `verify_child_no_cycles()` first and silently refuses
  (via `report_cycle()`) to introduce a graph cycle.
- **Stashing** (`stash_child()`/`unstash_child()`) moves a child to a
  separate list that is invisible, uncollidable, and skipped by normal
  traversal — distinct from `hide()` (draw-mask based, still collidable and
  still traversed) which lives on [NodePath](NodePath.md). Stash/unstash and
  the raw child-list mutators (`add_child`, `remove_child`, `stash_child(int)`
  by index, etc.) may only be called from pipeline stage 0 (the App thread).
- **Draw mask semantics (`adjust_draw_mask`)**: each node carries a
  `_draw_control_mask`/`_draw_show_mask` pair per-camera-bit.
  `show_mask` bits force show+controlled; `hide_mask` bits force hide+
  controlled; `clear_mask` bits clear "controlled" so that bit reverts to
  inheriting from the parent. Uncontrolled bits are implicitly "shown".
  This is the mechanism behind `NodePath::hide()`/`show()`/`show_through()`,
  which pass camera-specific masks down to this.
- **Bounding volumes come in three flavors**: *internal* (this node's own
  content, e.g. a GeomNode's geometry — computed by
  `compute_internal_bounds()`, overridden per node type), *external* (this
  node's internal bounds unioned with all children's external bounds — what
  `get_bounds()` returns), and *user* (an explicit override set via
  `set_bounds()`, which replaces the external bounds outright). Bounds and
  the derived `_nested_vertices`/`_net_collide_mask`/`_net_draw_*_mask`
  fields are cached and only recomputed when `_last_update != _next_update`
  (an `UpdateSeq` dirtied by `mark_bounds_stale()`).
- **State/effects/transform are the same interned objects described in the
  [module README](README.md#the-state-pipeline)** — `set_state()` etc.
  replace the node's slot with an already-interned `CPT()`, never mutate
  one in place.
- **Cull callback opt-in**: a node only gets `cull_callback()` invoked
  during traversal if it (or a subclass) called `set_cull_callback()` —
  tracked via the `FB_cull_callback` fancy bit — otherwise the traverser
  skips the virtual call entirely as an optimization.
- **`replace_node(other)`** swaps this node into the graph in place of
  `other`: copies transform/state/effects/tags, moves all children over,
  keeps this node at `other`'s position in its parent's child list, and —
  notably — **updates existing `NodePath`s that referenced `other`** to
  point at this node instead, rather than leaving them dangling. Used for
  "replace a node of one type with a node of a different type" operations
  (e.g. egg loader model-swap passes).
- **Pipelining**: `PandaNode`'s core data (`CData`, holding state/transform/
  children/parents/bounds/etc.) lives in a `PipelineCycler`, so a node
  mutated on the App thread mid-frame doesn't retroactively change what a
  render thread already reading it observes; `PandaNodePipelineReader`
  (below) is the read-only snapshot accessor used internally in
  performance-sensitive code (e.g. `CullTraverser`) instead of paying
  per-call cycler lookup overhead repeatedly.
- **`get_children()`/`get_stashed()`/`get_parents()`** return lightweight
  `Children`/`Stashed`/`Parents` snapshot proxies (copy-on-write shared
  pointer to the list) rather than requiring the lock to stay held across
  an iteration — prefer these over repeated `get_num_children()`/`get_child(n)`
  calls in a loop.

## API

**Construction / copying**
| Signature | Notes |
|---|---|
| `explicit PandaNode(const std::string &name)` | |
| `virtual PandaNode *make_copy() const` | Shallow copy, no children |
| `PT(PandaNode) copy_subgraph(Thread* = ...) const` | Deep copy of this node and all descendants |
| `virtual PandaNode *combine_with(PandaNode *other)` | Subclass hook to merge two nodes into one (used by flattening) |
| `PandaNode *dupe_for_flatten() const` | |

**Parent/child graph**
| Signature | Notes |
|---|---|
| `get_num_parents/get_parent(n)/find_parent` | |
| `get_num_children/get_child(n)/get_child_sort(n)/find_child` | |
| `Children get_children() const` / `Stashed get_stashed() const` / `Parents get_parents() const` | Snapshot proxies; index with `[]`, `.size()` |
| `add_child(child, sort=0)` | Re-adding replaces the previous instance; rejects cycles |
| `remove_child(index)` / `remove_child(node)` | |
| `replace_child(orig, new_child)` | |
| `stash_child(node)` / `stash_child(index)` | Moves to the stashed list (invisible + uncollidable + untraversed) |
| `unstash_child(node)` / `unstash_child(index)` | |
| `add_stashed(child, sort=0)` | Add directly as stashed |
| `remove_stashed(index)` | |
| `remove_all_children()` / `steal_children(other)` / `copy_children(other)` | |
| `count_num_descendants() const` | |

**Render state / effects / transform**
| Signature | Notes |
|---|---|
| `set_attrib(attrib, override=0)` / `get_attrib(type or slot)` / `has_attrib` / `clear_attrib` | Per-RenderAttrib-type shortcuts into the node's RenderState |
| `set_state(state)` / `get_state()` / `clear_state()` | Whole RenderState slot |
| `set_effect(effect)` / `get_effect(type)` / `has_effect` / `clear_effect` | |
| `set_effects(effects)` / `get_effects()` / `clear_effects()` | Whole RenderEffects slot |
| `set_transform(xform)` / `get_transform()` / `clear_transform()` | |
| `set_prev_transform` / `get_prev_transform` / `reset_prev_transform` / `has_dirty_prev_transform` | Previous-frame transform, for motion blur / vertex animation |

**Tags**
| Signature | Notes |
|---|---|
| `set_tag(key, value)` / `get_tag(key)` / `has_tag(key)` / `clear_tag(key)` | Arbitrary string key/value metadata on any node |
| `get_tag_keys(vector_string&)` / `get_num_tags()` / `get_tag_key(i)` | Enumerate |
| `copy_tags(other)` / `compare_tags(other)` / `list_tags(ostream&)` | |

**Visibility / collision masks**
| Signature | Notes |
|---|---|
| `adjust_draw_mask(show_mask, hide_mask, clear_mask)` | See Behavior notes |
| `get_draw_control_mask()` / `get_draw_show_mask()` | This node's own mask |
| `get_net_draw_control_mask()` / `get_net_draw_show_mask()` | Cached union across this node + descendants |
| `is_overall_hidden()` / `set_overall_hidden(bool)` | The special "overall" bit hides from every camera |
| `set_into_collide_mask(mask)` / `get_into_collide_mask()` / `get_legal_collide_mask()` | Which CollisionNode "from" masks can hit this node |
| `get_net_collide_mask()` | Cached union across descendants |
| `get_off_clip_planes()` | ClipPlaneAttrib representing clip planes turned off at/below this node |

**Bounds**
| Signature | Notes |
|---|---|
| `set_bounds_type(BoundsType)` / `get_bounds_type()` | Box vs. sphere vs. best/default |
| `set_bounds(volume)` / `set_bound(volume)` | Sets the *user* bounding volume override |
| `get_bounds()` / `get_bounds(UpdateSeq&)` | Returns the *external* bounds (cached) |
| `get_internal_bounds()` / `get_internal_vertices()` | This node's own content only |
| `get_nested_vertices()` | Vertex count, this node + descendants |
| `mark_bounds_stale()` / `mark_internal_bounds_stale()` / `is_bounds_stale()` | Force/check cache invalidation |
| `set_final(bool)` / `is_final()` | Treat external bounds as authoritative — stop culling children individually below this node |

**Type queries / misc**
| Signature | Notes |
|---|---|
| `is_geom_node()` / `is_lod_node()` / `is_collision_node()` / `as_light()` / `is_ambient_light()` | Cheap virtual type checks, avoids RTTI/dcast for hot paths |
| `is_scene_root()` / `is_under_scene_root()` | |
| `copy_all_properties(other)` | Copies transform/state/effects/tags/show-hide (not children) — see also `replace_node` |
| `replace_node(other)` | See Behavior notes — swaps `this` in for `other` in the graph, updating NodePaths |
| `set_unexpected_change(flags)` / `get_unexpected_change` / `clear_unexpected_change` | Debug tripwire: assert-fail if parents/children/transform/state/draw_mask change unexpectedly (`enum UnexpectedChange`) |
| `output(ostream&)` / `write(ostream&, indent)` / `ls(ostream&, indent)` | Debug dump |
| `static decode_from_bam_stream(data, reader=nullptr)` | Deserialize a single node from an in-memory bam blob |

### PandaNodeChain (internal, folded in)

**Source:** `panda/src/pgraph/pandaNodeChain.h/.I/.cxx`

A tiny private `LinkedListNode`-based linked list of `PandaNode`s, with no
public API. `PandaNode` uses one static instance
(`_dirty_prev_transforms`) to track which nodes currently have a
`_prev_transform` that differs from their current `_transform` (in
pipeline stage 0), so `PandaNode::reset_all_prev_transform()` can walk only
the dirty subset each frame instead of scanning every node in existence.

### PandaNodePipelineReader

A read-only, pipeline-stage-aware snapshot view of one `PandaNode`'s
`CData` (state, transform, children, parents, bounds, tags, masks — mirrors
most of `PandaNode`'s const accessors). Internal machinery
(`CullTraverser` and similar hot paths) constructs one of these per node
visited instead of repeatedly reacquiring the pipeline cycler per accessor
call. Not something application code typically constructs directly.

## See also

[NodePath](NodePath.md) — the handle application code actually uses;
[NodePathComponent](NodePathComponent.md) — per-instance path linkage stored
in this node's `_paths`; [RenderState](RenderState.md),
[RenderEffects](RenderEffects.md), [TransformState](TransformState.md) —
the state pipeline; [CullTraverser](CullTraverser.md) — consumes
`cull_callback()`/bounds/draw masks during rendering.
