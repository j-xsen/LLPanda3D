# GeomNode

**Source:** `panda/src/pgraph/geomNode.h` (+ `.I`, `.cxx`)
**Inherits:** PandaNode

A node that holds `Geom` objects — the primary kind of leaf node in the
scene graph; almost all visible objects are contained in a `GeomNode`
somewhere. `Geom` itself (a piece of renderable geometry: primitives +
vertex data) is defined in `panda/src/gobj` (undocumented here) —
`GeomNode` is the scene-graph attachment point for it, not the geometry
representation itself.

## Behavior notes

- Each entry is a `(COWPT(Geom) geom, CPT(RenderState) state)` pair
  (`GeomEntry`). The per-Geom `RenderState` is composed with whatever
  state has accumulated from ancestor nodes during cull — it does *not*
  replace the inherited scene-graph state, it layers on top of it. Pass
  `RenderState::make_empty()` (the `add_geom()` default) to inherit purely
  from the graph.
- The Geom list itself is a **pipeline-cycled, copy-on-write** structure
  (`CopyOnWriteObj<pvector<GeomEntry>>` wrapped in `PipelineCycler`).
  `get_geom(n)` returns a read pointer (don't modify — may be shared with
  other GeomNodes); `modify_geom(n)` triggers copy-on-write so the
  returned `Geom` is unique to this node, and marks internal bounds stale.
  **Caution called out in source:** calling `modify_geom()`/`set_geom_state()`
  during a downstream pipeline stage (cull/draw) propagates the change all
  the way back to stage 0, potentially clobbering independent stage-0
  changes made the same frame.
- `get_geoms(thread)` returns a `GeomNode::Geoms` helper — a cached
  snapshot of the geom list, faster than repeated `get_geom(n)` calls when
  visiting all geoms, and safe against self-modifying loops (adding/
  removing geoms mid-traversal) since it holds its own COW reference.
- `set_preserved(true)` opts a node out of `safe_to_flatten()` and
  `safe_to_combine()` — the scene graph flattener (`SceneGraphReducer`)
  will leave it alone entirely.
- `combine_with(other)`: two exact-type `GeomNode`s combine by literally
  moving all Geoms from `other` into `this` via `add_geoms_from()`
  (`decompose()`/`unify()` are separate, explicit operations — combining
  nodes doesn't automatically merge their individual Geoms into fewer
  primitives).
- `add_for_draw()` (called during cull) composes each Geom's state with
  the traversal's accumulated state, invokes any `has_cull_callback()` on
  the composed state (letting a state veto rendering of that Geom), and —
  **only when the node has more than one Geom** — additionally culls each
  individual Geom's bounding volume against the view frustum and cull
  planes (skipped for single-Geom nodes since the node's own bounding
  volume, already tested by the caller, is presumed to equal the one
  Geom's volume). Surviving Geoms become `CullableObject`s handed to the
  `CullHandler`.
- `get_legal_collide_mask()` returns `CollideMask::all_on()` — unlike most
  `PandaNode` subclasses (which return 0, meaning "can't be collided
  with"), `GeomNode` (like `CollisionNode`) is a valid collision target.
- `decompose()` breaks composite primitives (triangle strips/fans) down to
  indexed triangles on every contained Geom — normally only useful as
  prep before `unify()`; `SceneGraphReducer::decompose()` is the usual
  entry point rather than calling this directly.
- `unify(max_indices, preserve_order)` greedily merges Geoms that share
  the same `RenderState` (an O(n²) pass, fine since node Geom counts are
  small) by calling `Geom::copy_primitives_from()`, then unifies each
  surviving Geom's own primitives via `Geom::unify_in_place()`. Merge only
  succeeds when primitives reference the same `GeomVertexData`, share
  fundamental primitive type, and have compatible shade models.
  `preserve_order=true` restricts merge attempts to the tail of the
  output list only, trading optimality for stable primitive ordering.
- `write_geoms()`/`write_verbose()` are debug dumps — the latter also
  writes each Geom's own internal structure.

## API

| Method | Notes |
|---|---|
| `GeomNode(name)` | Constructor |
| `set_preserved(bool)` / `get_preserved()` | Opt out of flatten/combine |
| `get_num_geoms()` | |
| `get_geom(n)` / `modify_geom(n)` | Read pointer vs. copy-on-write pointer |
| `get_geom_state(n)` / `set_geom_state(n, RenderState*)` | Per-Geom local state |
| `get_geoms(thread)` → `Geoms` | Snapshot iterator, faster for bulk visits |
| `add_geom(Geom*, state = make_empty())` | Append a Geom |
| `add_geoms_from(GeomNode*)` | Copy all Geoms (+states) from another node |
| `set_geom(n, Geom*)` | Replace an existing slot |
| `remove_geom(n)` / `remove_all_geoms()` | |
| `check_valid()` | Validates every contained Geom |
| `decompose()` | Break composite primitives into indexed triangles on every Geom |
| `unify(max_indices, preserve_order)` | Merge compatible Geoms/primitives to minimize count |
| `write_geoms(ostream&, indent)` / `write_verbose(ostream&, indent)` | Debug dumps |
| `get_default_collide_mask()` (static) | Default `into_collide_mask` for new GeomNodes |

## Usage

```cpp
PT(GeomNode) gnode = new GeomNode("model_geom");
gnode->add_geom(my_geom, RenderState::make_empty());
NodePath gnp = parent.attach_new_node(gnode);
```

## See also

- [PandaNode](PandaNode.md) — base class
- [RenderState](RenderState.md) — per-Geom local state
- [SceneGraphReducer](SceneGraphReducer.md) — typical caller of `decompose()`/flattening
- `Geom`, `GeomVertexData` — `panda/src/gobj` (undocumented)
