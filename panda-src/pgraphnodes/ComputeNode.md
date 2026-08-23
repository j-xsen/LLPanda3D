# ComputeNode

**Source:** `panda/src/pgraphnodes/computeNode.h` (+ `.I`, `.cxx`)
**Inherits:** `PandaNode`
**Inherited by:** (none)

A node whose sole purpose is to invoke a dispatch operation on a compute
shader. It carries no geometry — attach a `Shader` compiled in compute mode
via a `ShaderAttrib` (see [../gobj/Shader.md](../gobj/Shader.md)) to this
node (or an ancestor), then add one or more dispatch commands specifying
the work-group counts.

## Behavior notes

- **Dispatch is issued through an internal `Dispatcher`.** `ComputeNode`
  itself follows the same draw-callback pattern documented in
  [CallbackNode.md](CallbackNode.md), but it's not a `CallbackNode`
  subclass — `add_for_draw()` always queues a `CullableObject` with a
  private `ComputeNode::Dispatcher` (a `CallbackObject` subclass) as the
  draw callback. `Dispatcher::do_callback()` casts the `CallbackData` to a
  `GeomDrawCallbackData`, gets the current GSG from it, and calls
  `gsg->dispatch_compute(x, y, z)` once per stored dispatch command, in
  the order they were added.
- **Multiple dispatches, one node.** `add_dispatch()` can be called more
  than once; each call queues an additional `(x, y, z)` work-group triple,
  all issued in sequence when the node is drawn — useful for chained
  compute passes that all use the same bound shader/buffers.
- Like `CallbackNode`, the constructor sets an infinite
  `OmniBoundingVolume` (never frustum-culled) and `safe_to_combine()`
  returns `false`.
- The `Dispatcher::CData::get_parent_type()` override returns
  `CallbackNode::get_class_type()` rather than `ComputeNode`'s own type —
  this looks like it may be copy-pasted from `CallbackNode`'s equivalent
  `CData` and is purely a Bam-versioning parent-type tag for `CycleData`;
  it doesn't imply `ComputeNode` or its `Dispatcher` actually inherit from
  `CallbackNode` (they don't — see Inherits above).

## API

| Method | Notes |
|---|---|
| `ComputeNode(const std::string &name)` | Construct with a node name. Assign a compute `Shader` via `ShaderAttrib` separately. |
| `add_dispatch(const LVecBase3i &num_groups)` / `add_dispatch(int x, int y, int z)` | Append a dispatch command (work-group counts per dimension; use 1 for unused dimensions). |
| `get_num_dispatches() const` | Number of queued dispatch commands. |
| `get_dispatch(size_t i) const` / `set_dispatch(size_t i, ...)` | Read/overwrite the i'th dispatch command. |
| `insert_dispatch(size_t i, ...)` / `remove_dispatch(size_t i)` | Insert/remove a dispatch command by index. |
| `clear_dispatches()` | Remove all dispatch commands. |
| `dispatches` (`MAKE_SEQ_PROPERTY`) | Python-side sequence view over the dispatch list. |

## Usage

```cpp
PT(ComputeNode) node = new ComputeNode("particle_update");
node->add_dispatch(LVecBase3i(64, 1, 1));
NodePath np = parent.attach_new_node(node);
np.set_shader(compute_shader);
```

## See also

- [CallbackNode](CallbackNode.md) — the general-purpose draw/cull callback
  node this class's dispatch mechanism parallels
- [../gobj/Shader.md](../gobj/Shader.md),
  [../gobj/ShaderBuffer.md](../gobj/ShaderBuffer.md) — the compute shader
  program and any storage buffers it reads/writes
