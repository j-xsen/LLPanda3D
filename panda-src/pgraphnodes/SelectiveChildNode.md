# SelectiveChildNode

**Source:** `panda/src/pgraphnodes/selectiveChildNode.h` (+ `.I`, `.cxx`)
**Inherits:** `PandaNode` **Inherited by:** [SwitchNode](SwitchNode.md), [SequenceNode](SequenceNode.md)

Abstract base for nodes that render only a subset (typically exactly one)
of their children rather than all of them — [SwitchNode](SwitchNode.md)
(explicit index) and [SequenceNode](SequenceNode.md) (time-cycled).
[LODNode](LODNode.md) is conceptually similar (also selects one child) but
does **not** inherit `SelectiveChildNode` — it implements its child
selection directly in `cull_callback()` instead of through this class's
`get_first_visible_child()`/`get_next_visible_child()` protocol.

## Behavior notes

- **`has_selective_visibility()` returning `true` changes how cull
  traversal enumerates children.** Instead of iterating every child in
  order, the cull traverser calls `get_first_visible_child()` then
  repeatedly `get_next_visible_child(n)` until it returns a value
  `>= get_num_children()`. This method is called *after* `cull_callback()`,
  so `cull_callback()` is responsible for deciding which child(ren) should
  be visible (via `select_child()`) before traversal asks which ones those
  are.
- **The base class implementation is a single-child protocol**:
  `get_first_visible_child()` returns `_selected_child` and
  `get_next_visible_child()` always returns `get_num_children()` (i.e. "no
  more") — so by default exactly one child (whichever `select_child()` last
  set) is visible. Subclasses needing genuinely multiple visible children
  would need to override both.
- **`_selected_child` is a plain (non-pipeline-cycled) member**, unlike most
  mutable `PandaNode` state. A source comment flags this explicitly as not
  fully thread-safe and calls it "probably a problem in the non-thread-safe
  design" — worth knowing if you're debugging pipelining/threading issues
  around these nodes.
- `select_child()` is `protected` — only subclasses call it (from their own
  `cull_callback()`), not application code directly.

## API

| Method | Notes |
|---|---|
| `SelectiveChildNode(name)` | Constructor |
| `has_selective_visibility()` | Virtual; base returns `true` |
| `get_first_visible_child()` / `get_next_visible_child(n)` | Virtual; base implements single-child selection via `_selected_child` |
| `select_child(n)` *(protected)* | Sets `_selected_child`; called by subclasses' `cull_callback()` |

## See also

- [SwitchNode](SwitchNode.md), [SequenceNode](SequenceNode.md) — concrete subclasses
- [LODNode](LODNode.md) — a similar "one visible child" node that does *not* use this base
