# CallbackNode

**Source:** `panda/src/pgraphnodes/callbackNode.h` (+ `.I`, `.cxx`)
**Inherits:** `PandaNode`
**Inherited by:** (none)

A node that issues an arbitrary user-supplied `CallbackObject` callback
during the cull traversal, the draw traversal, or both, instead of (or in
addition to) rendering geometry. Use it to hook custom logic — debug
visualization, procedural state changes, non-`Geom` draw calls — directly
into the render pipeline at a specific point in the scene graph.

## Behavior notes

- **The cull callback *replaces* normal cull behavior by default.** If
  `set_cull_callback()` is set and the callback does nothing else, the cull
  traversal stops at this node — its children are **not** visited. To get
  the normal "keep traversing" behavior, the callback must explicitly call
  `NodeCullCallbackData::upcall()` (see below), which both queues this
  node's draw callback (if any) and manually re-invokes the traverser on
  each child.
- **Infinite default bounding volume.** The constructor sets
  `set_internal_bounds(new OmniBoundingVolume)` — a `CallbackNode` is never
  view-frustum-culled by default, on the theory that a naive user who never
  sets a real bounding volume should still see their callback fire rather
  than have it silently culled. Set a tighter bounding volume yourself if
  the callback's effect is spatially local.
- **`safe_to_combine()` returns `false`** — like `LODNode`, a `CallbackNode`
  is never merged with siblings during scene graph flattening, since its
  identity (and thus which callback fires when) is meaningful.
- The draw callback is passed a `GeomDrawCallbackData` (documented in
  `pgraph`'s `Geom`/cull-pipeline material, see
  [../pgraph/README.md](../pgraph/README.md#cull-pipeline)), **not** a
  `NodeCullCallbackData` — the two callback data types differ because the
  cull and draw traversals run on different threads with different
  available context (cull has the `CullTraverser`/scene-graph position;
  draw has the current GSG/state/transform). The draw callback's `Geom`
  pointer is always `nullptr`, since `CallbackNode` doesn't carry any
  `Geom`s of its own.

## API

| Method | Notes |
|---|---|
| `CallbackNode(const std::string &name)` | Construct with a node name. |
| `set_cull_callback(CallbackObject *object)` / `clear_cull_callback()` / `get_cull_callback()` | Callback invoked (on the cull thread) once this node passes the bounding-volume/frustum test. |
| `set_draw_callback(CallbackObject *object)` / `clear_draw_callback()` / `get_draw_callback()` | Callback invoked (on the draw thread) when this node's queued `CullableObject` is actually drawn. |

### `NodeCullCallbackData` (folded in — `nodeCullCallbackData.h`)

The `CallbackData` subclass (inherits the external, undocumented `putil`
class `CallbackData`) passed to a `CallbackNode`'s cull callback.

| Method | Notes |
|---|---|
| `get_trav() const` | The `CullTraverser` in use — traversal-invariant data (`DisplayRegion`, `Camera`, …). |
| `get_data() const` | The `CullTraverserData` for *this* node — current position in the traversal, accumulated transform/state. |
| `upcall()` | Restores default behavior: if this node is a `CallbackNode` with a draw callback set, queues a `CullableObject` for it; then manually re-invokes `trav->traverse()` on every child. Call this from your cull callback if you want traversal to continue normally below this node. |

## Usage

```cpp
PT(CallbackNode) node = new CallbackNode("debug_hook");
node->set_draw_callback(new MyDrawCallback());
NodePath np = parent.attach_new_node(node);
```

## See also

- [ComputeNode](ComputeNode.md) — a separate `PandaNode` subclass (not a
  `CallbackNode` subclass) that follows the same "install a draw callback
  to get queued at draw time" pattern, using an internal `CallbackObject`
  (`Dispatcher`) to issue a compute-shader dispatch instead of app-supplied
  code
- `../pgraph/README.md#cull-pipeline` — `CullTraverser`/`CullableObject`/
  `CullHandler` context these callbacks run inside
