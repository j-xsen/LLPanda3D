# DataGraphTraverser

**Source:** `panda/src/dgraph/dataGraphTraverser.h` / `.I` / `.cxx`
**Inherits:** *(standalone class, not a `TypedObject`)*
**Operates on:** [DataNode](DataNode.md) trees, moving [DataNodeTransmit](DataNodeTransmit.md) between them

`DataGraphTraverser` is a stack-allocated helper, not a persistent object —
construct one, call `traverse()` on a root `PandaNode`, and let it go out of
scope. It walks the tree once, transmitting data from each `DataNode` to its
`DataNode` children.

## Behavior notes

- **`traverse()` requires the given node to be an actual root if it's a
  `DataNode`.** `nassertv(data_node->get_num_parents(...) == 0)` — passing a
  `DataNode` that has parents is a programming error caught by assertion.
  Passing a non-`DataNode` node is fine regardless of its parentage; the
  traverser just recurses into its children with empty input.
- **A `DataNode` with exactly one `DataNode` parent transmits immediately**;
  one with 0 or 2+ `DataNode` parents goes through a different path.
  0-`DataNode`-parent nodes (reached e.g. as an immediate child of the
  traversal root, or below a non-`DataNode` node) transmit immediately too,
  with an empty input.
- **Multi-parent `DataNode`s are collected across the whole traversal, not
  resolved locally.** A child with N `DataNode` parents is stashed in
  `_multipass_data` (keyed by node pointer) the first time it's reached, and
  only actually transmitted once data has arrived from all N — via
  `CollectedData::set_data()`, indexed by `find_parent()`'s result, so the
  order the traverser happens to visit parents in doesn't matter, only that
  all of them are eventually visited within this same `traverse()` call.
- **`collect_leftovers()` (called automatically at the end of `traverse()`)
  force-transmits any node that never got all its expected parent data** —
  this happens when a `DataNode` has a parent that lies outside the subtree
  actually passed to `traverse()`/`traverse_below()`. It logs
  `dgraph_cat.warning() << *data_node << " improperly parented partly
  outside of data graph."` and transmits with whatever partial data (empty
  `DataNodeTransmit()` slots for the missing parents) was collected.
- **Non-`DataNode` nodes are transparent to data flow, not blockers.**
  `traverse_below()` recurses through any child that isn't itself a
  `DataNode`, re-passing the same `DataNodeTransmit` output further down —
  so a `DataNode` several levels below a plain `PandaNode` in the hierarchy
  can still receive data from an ancestor `DataNode`, as long as a
  `DataNode` descendant is eventually reached to consume it by name/type.
- **One `DataGraphTraverser` targets one `Thread`.** The constructor captures
  `current_thread` (defaulting to `Thread::get_current_thread()`) and uses it
  for every `get_children()`/`get_num_parents()` call during the traversal —
  relevant if driving the data graph from a non-App thread.

## API

| Signature | Notes |
|---|---|
| `explicit DataGraphTraverser(Thread *current_thread = Thread::get_current_thread())` | |
| `Thread *get_current_thread() const` | |
| `void traverse(PandaNode *node)` | Full traversal from `node`: transmits `node` itself (if a `DataNode`, asserting it has no parents) or recurses into its children (if not), then calls `collect_leftovers()` |
| `void traverse_below(PandaNode *node, const DataNodeTransmit &output)` | Continues the traversal into `node`'s children, feeding them `output` — does **not** call `transmit_data()` on `node` itself; used internally by `traverse()` and recursively by itself |
| `void collect_leftovers()` | Force-transmits any node left incomplete in `_multipass_data`; called automatically at the end of `traverse()`, but exposed publicly for callers that drive traversal manually via `traverse_below()` |

## Usage

```cpp
// Typical per-frame driver (roughly what ShowBase does internally):
NodePath data_root("data_root");
PT(MouseAndKeyboard) mak = new MouseAndKeyboard(win, 0, "mak");
data_root.attach_new_node(mak);
// ... attach Trackball, MouseWatcher, etc. as DataNode children of mak ...

DataGraphTraverser trav;
trav.traverse(data_root.node());  // call once per frame
```

## See also

[README.md](README.md) for the shared data-graph model ·
[DataNode](DataNode.md) (the nodes being traversed) ·
[DataNodeTransmit](DataNodeTransmit.md) (the payload moved between them)
