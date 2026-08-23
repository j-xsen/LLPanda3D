# NodePathComponent

**Source:** `panda/src/pgraph/nodePathComponent.h` (+ `.I`, `.cxx`)
**Inherits:** ReferenceCount

One link in the singly-linked chain that a [NodePath](NodePath.md) walks
from a node up to the scene graph root. Every `PandaNode` lazily allocates
one `NodePathComponent` per distinct parent it's instanced under (never one
per `NodePath` — many `NodePath`s through the same graph region share the
same chain of components, ref-counted). Application code never constructs
one directly; `PandaNode` creates and manages them via its private
`get_component()`/`get_top_component()`/`delete_component()` methods, which
`NodePath` and [WorkingNodePath](WorkingNodePath.md) call as friends.

## Behavior notes

- **Pipelined `_next`:** the pointer to the next component up the chain is
  stored in a `CycleData`/`PipelineCycler`, not a plain member — reparenting
  a node changes a component's `_next` per pipeline stage so that a render
  thread mid-frame keeps seeing the graph shape it started with. `_node`
  and `_key` are NOT cycled — they're permanent for the component's
  lifetime (the key becomes permanent once first requested, since key `0`
  means "unassigned").
- **Keys are lazily generated and globally unique for the process lifetime**
  (`get_key()`), guarded by a static `_key_lock`. Generated on first
  request rather than at construction, to avoid burning through the 32-bit
  key space for components nobody ever asks to identify.
- **`fix_length()`** repairs a cached path-length counter (`_length`) if it
  drifts out of sync with its predecessor's length + 1 — called after
  graph-structure changes that might invalidate it.
- Destroying a component calls back into its owning `PandaNode::delete_component()`
  to unregister itself — components don't outlive their node silently.

## API

| Method | Notes |
|---|---|
| `get_node() const` | The `PandaNode` this component represents |
| `has_key() const` | Whether `get_key()` has already generated a key |
| `get_key() const` | Globally unique, permanent int id (generated lazily) |
| `is_top_node(stage, thread) const` | True if this component has no `_next` (it's a graph root) |
| `get_next(stage, thread) const` | The parent component, or `nullptr` at the root |
| `get_length(stage, thread) const` | Cached depth of the path to this component |
| `fix_length(stage, thread)` | Recomputes `_length` if stale; returns whether it changed |
| `output(ostream&) const` | Writes `parent/parent/.../this`, marking stashed nodes with `@@` |

## See also

[NodePath](NodePath.md), [PandaNode](PandaNode.md), [WeakNodePath](WeakNodePath.md), [WorkingNodePath](WorkingNodePath.md)
