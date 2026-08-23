# CollisionTraverser

**Source:** `panda/src/collide/collisionTraverser.h` (+ `.I`, `.cxx`)
**Inherits:** `Namable`

"Manages the traversal through the scene graph to detect collisions." Owns a
set of `(CollisionNode NodePath, CollisionHandler)` pairs; `traverse(root)`
walks the graph once from `root` down, finds every solid pair whose bounds
overlap, runs the real per-shape test, and hands each confirmed hit to the
appropriate [CollisionHandler](CollisionHandler.md).

## Behavior notes

- **One `traverse()` call tests *all* registered colliders together in a
  single recursive walk, not once per collider.** Internally this uses
  `CollisionLevelState<MaskType>` (from `collisionLevelState.h`, not
  separately documented — see the exclusions note in [README.md](README.md)):
  a per-recursion-level bitmask tracking which colliders are still
  "in play" for the current node, so a collider whose bounding volume
  doesn't reach a subtree gets pruned from further descent into it. Three
  instantiations exist for different collider counts —
  `CollisionLevelStateSingle`/`Double`/`Quad` (32/64/128 colliders) — and by
  default the traverser always picks the narrowest one that fits, unless
  `allow-collider-multiple` (Config.prc) is set, in which case it may
  additionally run separate narrower passes for different collider-count
  buckets.
- **`add_collider(collider, handler)` requires `collider.node()->is_collision_node()`**
  and asserts if `handler` is null. Re-adding the same NodePath just retargets
  it to a new handler (doesn't duplicate the registration). One handler may
  serve many colliders; a given `CollisionNode` is served by exactly one
  handler at a time.
- **Collider ordering is driven by `CollisionNode::get_collider_sort()`,
  ascending, not registration order.** See [CollisionNode.md](CollisionNode.md).
- **`set_respect_prev_transform(true)` enables swept/continuous collision** —
  see the same-named note in [README.md](README.md)'s "Core concepts". Off
  by default (mirrors the `respect-prev-transform` Config.prc variable at
  construction time, but is a per-traverser flag afterward).
- **Debug recording/visualization is compiled in only if `DO_COLLISION_RECORDING`
  is defined** (the usual default for non-release builds). `show_collisions(root)`
  attaches a [CollisionVisualizer](CollisionVisualizer.md) under `root` and
  installs it as this traverser's recorder in one call; `hide_collisions()`
  tears it back down. `set_recorder()` lets you install a custom
  [CollisionRecorder](CollisionRecorder.md) subclass instead.
- **No default handler is auto-assigned** — every collider you add must
  come with an explicit `CollisionHandler*`. (`_default_handler` exists
  internally but is only used as a fallback in specific internal code paths,
  not something application code relies on.)

## API

### Colliders
| Signature | Notes |
|---|---|
| `explicit CollisionTraverser(const std::string &name = "ctrav")` | |
| `void add_collider(const NodePath &collider, CollisionHandler *handler)` | |
| `bool remove_collider(const NodePath &collider)` / `bool has_collider(const NodePath&) const` / `void clear_colliders()` | |
| `int get_num_colliders() const` / `NodePath get_collider(int) const` | |
| `CollisionHandler *get_handler(const NodePath &collider) const` | |

### Traversal
| Signature | Notes |
|---|---|
| `BLOCKING void traverse(const NodePath &root)` | Runs one full pass; call once per frame, typically from a task |
| `void set_respect_prev_transform(bool)` / `bool get_respect_prev_transform() const` | Swept collision toggle |

### Debugging (`DO_COLLISION_RECORDING` builds only)
| Signature | Notes |
|---|---|
| `void set_recorder(CollisionRecorder*)` / `bool has_recorder() const` / `CollisionRecorder *get_recorder() const` / `void clear_recorder()` | |
| `CollisionVisualizer *show_collisions(const NodePath &root)` / `void hide_collisions()` | Convenience wrapper around a `CollisionVisualizer` recorder |

## Usage

```cpp
CollisionTraverser ctrav("main-ctrav");
PT(CollisionHandlerQueue) queue = new CollisionHandlerQueue();
ctrav.add_collider(picker_ray_np, queue);

// each frame:
ctrav.traverse(render);
queue->sort_entries();
```

## See also

[CollisionNode.md](CollisionNode.md) · [CollisionHandler.md](CollisionHandler.md)
· [CollisionEntry.md](CollisionEntry.md) · [CollisionRecorder.md](CollisionRecorder.md)
· [CollisionVisualizer.md](CollisionVisualizer.md) · [README.md](README.md)
