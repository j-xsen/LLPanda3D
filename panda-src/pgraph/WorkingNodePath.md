# WorkingNodePath

**Source:** `panda/src/pgraph/workingNodePath.h` (+ `.I`, `.cxx`)

A low-overhead stand-in for a [NodePath](NodePath.md) used during full
scene-graph traversals (e.g. [FindApproxPath](FindApproxPath.md) searches).
A real `NodePath` allocates and links a
[NodePathComponent](NodePathComponent.md) on every node it touches, and
unlinks it on destruction; walking a whole graph that way is expensive.
`WorkingNodePath` instead forms an in-memory linked list purely on the
stack/heap of the traversal itself — each step wraps its parent
`WorkingNodePath` plus one child `PandaNode*`, with no `PandaNode`-side
bookkeeping — and only materializes a real `NodePath` (via
`get_node_path()`) on demand, e.g. once a search match is found.

## Behavior notes

- Trades consistency for speed: since it doesn't register components on the
  nodes it passes through, it does **not** guarantee correctness if the
  scene graph is mutated while the traversal is in progress.
- `get_node_path()` walks back up the `_next` chain, calling
  `PandaNode::get_component()` at each level to build a real, properly
  linked `NodePathComponent` chain. If it discovers the chain became
  disconnected partway (an ancestor was reparented mid-traversal), it
  truncates: `PandaNode::get_top_component()` is used at the break point
  instead of failing outright.
- Two internal representations coexist in one object: either `_start` is
  set (this is the root, wrapping an existing `NodePathComponent`) or
  `_next` is set (this is one step down from a parent `WorkingNodePath`) —
  never both.

## API

| Method | Notes |
|---|---|
| `WorkingNodePath(const NodePath &start)` | Begin a traversal at an existing NodePath |
| `WorkingNodePath(const WorkingNodePath &parent, PandaNode *child)` | Extend the traversal one node down |
| `is_valid() const` | Sanity-checks the chain is well-formed |
| `node() const` | The `PandaNode*` at this step |
| `get_node_path() const` | Materializes a real `NodePath` for this point in the traversal |
| `get_num_nodes() const` | Depth of the path so far (root through this node, ≥ 2) |
| `get_node(int index) const` | Node at `index` (0 = this node, increasing toward the root) |

## See also

[NodePath](NodePath.md), [NodePathComponent](NodePathComponent.md), [FindApproxPath](FindApproxPath.md)
