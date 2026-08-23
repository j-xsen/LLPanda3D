# WeakNodePath

**Source:** `panda/src/pgraph/weakNodePath.h` (+ `.I`, `.cxx`)

A non-owning handle to a [NodePath](NodePath.md). A regular `NodePath`
holds a reference-counted pointer chain, so as long as any `NodePath`
exists the referenced nodes can't be destructed even if fully detached from
the scene graph; `WeakNodePath` wraps a `WeakPointerTo<NodePathComponent>`
instead, so the node may be deleted out from under it at any time. Useful
for caches, listener registries, or any place that needs to remember "this
node, if it still exists" without keeping it alive.

## Behavior notes

- `was_deleted()` and `is_empty()` differ: `is_empty()` is true for both "no
  node was ever assigned" and "the node was deleted"; `was_deleted()` is
  true only for the latter (a real reference existed and was cleared out
  from under this weak handle).
- `get_node_path()` returns an empty `NodePath` with its error flag set
  (`ET_fail`) if the node was deleted — check `was_deleted()` first, or
  check the returned `NodePath`'s validity, rather than assuming a valid
  result.
- Equality/ordering (`operator==`, `operator<`, `compare_to`) work even
  against a deleted node, by comparing the underlying weak-pointer identity
  (`owner_before`) — so a `WeakNodePath` remains usable as a map/set key
  after its node is gone.

## API

| Method | Notes |
|---|---|
| `WeakNodePath(const NodePath&)` | Wrap a NodePath without taking ownership |
| `operator=` | Reassign from a `NodePath` or another `WeakNodePath` |
| `clear()` | Reset to empty |
| `operator bool() const` | True if pointing to a valid, non-deleted node |
| `is_empty() const` | No node assigned, or node was deleted |
| `was_deleted() const` | Node was assigned but has since been deleted |
| `get_node_path() const` | Reconstructs a real `NodePath` (error-flagged if deleted) |
| `node() const` | Returns the `PandaNode*` directly, or `nullptr` if deleted |
| `get_key() const` | Same identity key as `NodePath::get_key()`, cached across deletion |
| `output(ostream&) const` | Writes the path, or `"deleted"` |

## Usage

```cpp
WeakNodePath weak_ref(some_node_path);
// ... later, possibly after the node was removed from the scene graph ...
if (weak_ref) {
  NodePath np = weak_ref.get_node_path();
  np.set_color(1, 0, 0, 1);
} else if (weak_ref.was_deleted()) {
  // node is gone; drop this cache entry
}
```

## See also

[NodePath](NodePath.md), [NodePathComponent](NodePathComponent.md)
