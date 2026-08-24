# CollisionHandlerQueue

**Source:** `panda/src/collide/collisionHandlerQueue.h` (+ `.cxx`)
**Inherits:** [CollisionHandler](CollisionHandler.md)

"Does nothing except remember the CollisionEntries detected the last pass."
No events, no physics — a flat list polled afterward.
"It's primarily useful when a simple intersection test is being made, e.g.
for picking from the window" — the standard mouse-picking handler, paired
with a [CollisionRay](CollisionRay.md).

## Behavior notes

- **`begin_group()` clears the list; entries accumulate via `add_entry()`
  until the next `begin_group()`.** So after `traverse()`, the queue holds
  exactly this pass's hits — no in/again/out diffing like
  [CollisionHandlerEvent](CollisionHandlerEvent.md).
- **Entries are in arbitrary (traversal) order until `sort_entries()` is
  called**, which orders them front-to-back by distance from the
  *from* solid's origin — call it before assuming `get_entry(0)` is the
  closest hit (the standard mouse-picking pattern).

## API

| Signature | Notes |
|---|---|
| `CollisionHandlerQueue()` | |
| `void sort_entries()` | Nearest-first ordering by distance from the from-solid's origin |
| `void clear_entries()` | Manual reset |
| `int get_num_entries() const` / `CollisionEntry *get_entry(int) const` | |

## Usage

```cpp
PT(CollisionRay) pray = new CollisionRay();
PT(CollisionNode) pnode = new CollisionNode("picker");
pnode->add_solid(pray);
pnode->set_from_collide_mask(CollideMask::bit(1));
pnode->set_into_collide_mask(CollideMask(0));
NodePath picker_np = camera.attach_new_node(pnode);

PT(CollisionHandlerQueue) queue = new CollisionHandlerQueue();
ctrav.add_collider(picker_np, queue);

// each frame, after positioning pray via set_from_lens():
ctrav.traverse(render);
queue->sort_entries();
if (queue->get_num_entries() > 0) {
  NodePath hit = queue->get_entry(0)->get_into_node_path();
}
```

## See also

[CollisionHandler.md](CollisionHandler.md) · [CollisionRay.md](CollisionRay.md)
· [CollisionEntry.md](CollisionEntry.md) · [README.md](README.md)
