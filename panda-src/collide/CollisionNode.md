# CollisionNode

**Source:** `panda/src/collide/collisionNode.h` (+ `.I`, `.cxx`)
**Inherits:** `PandaNode`

"A node in the scene graph that can hold any number of
[CollisionSolid](CollisionSolid.md)s." Either static geometry that other
things collide *into*, or an animated collider that moves around detecting
things — the same class serves both roles; which role(s) it plays is
controlled by its collide masks.

## Behavior notes

- **Two independent collide masks: `from_collide_mask` (this node's own,
  not inherited) and `into_collide_mask` (inherited from `PandaNode`,
  shared with ordinary renderable nodes).** A pair `(A, B)` is only tested
  by [CollisionTraverser](CollisionTraverser.md) if
  `(A.get_from_collide_mask() & B.get_into_collide_mask()) != 0`. Both
  default to `get_default_collide_mask()` — the low 20 bits
  (`CollideMask::lower_on(20)`, from `panda/src/putil/collideMask.h`).
  Ordinary `GeomNode`s default their `into_collide_mask` to bit 20
  (`default_geom_node_collide_mask`) — one bit above the reserved
  `CollisionNode` range, unassigned by default, so a fresh `CollisionNode`
  won't accidentally collide against plain scene geometry until you opt in
  by widening its `from_collide_mask` to include bit 20 (or higher, for
  your own custom bits). `set_collide_mask(mask)` sets both at once.
- **`get_legal_collide_mask()` returns `CollideMask::all_on()`** — unlike a
  `GeomNode`, a `CollisionNode` is not restricted to a narrower legal range.
- **`collider_sort` controls traversal ordering among colliders sharing one
  [CollisionTraverser](CollisionTraverser.md), lower first.** Useful when one
  collider's handler depends on another having already been processed this
  frame (e.g. a floor-follower should generally run before a pusher that
  reads the post-floor position). Set via `set_collider_sort()`; only
  meaningful when the node is registered as a *from* collider via
  `CollisionTraverser::add_collider()`.
- **Not renderable in the normal sense** (`is_renderable()` distinguishes it
  from a `GeomNode`), but `cull_callback()` still draws its solids'
  visualization geometry when collision-solid display is enabled (`show-collision-solids`
  in Config.prc, or toggling via `NodePath.show()`/framework debug keys).
- **`combine_with(other)` merges two `CollisionNode`s only if their name,
  transform, and *both* collide masks all match exactly** — otherwise the
  scene graph flattener leaves them separate. Also gated globally by the
  `flatten-collision-nodes` config variable.

## API

### Construction
| Signature | Notes |
|---|---|
| `explicit CollisionNode(const std::string &name)` | |

### Collide masks
| Signature | Notes |
|---|---|
| `void set_collide_mask(CollideMask)` | Sets both from- and into-mask |
| `void set_from_collide_mask(CollideMask)` / `CollideMask get_from_collide_mask() const` | This node's own "from" mask |
| `void set_into_collide_mask(CollideMask)` / `CollideMask get_into_collide_mask() const` | Inherited-style "into" mask (via `PandaNode`) |
| `static CollideMask get_default_collide_mask()` | Low 20 bits |

### Solids
| Signature | Notes |
|---|---|
| `size_t add_solid(const CollisionSolid*)` | Returns the new index |
| `size_t get_num_solids() const` / `CPT(CollisionSolid) get_solid(size_t) const` / `PT(CollisionSolid) modify_solid(size_t)` | `modify_solid()` triggers copy-on-write |
| `void set_solid(size_t, CollisionSolid*)` / `insert_solid(size_t, const CollisionSolid*)` / `remove_solid(size_t)` / `clear_solids()` | |

### Ordering
| Signature | Notes |
|---|---|
| `int get_collider_sort() const` / `void set_collider_sort(int)` | Lower runs first among this traverser's colliders |

## Usage

```cpp
PT(CollisionNode) cnode = new CollisionNode("player-collider");
cnode->add_solid(new CollisionSphere(0, 0, 0, 1.0));
cnode->set_from_collide_mask(CollideMask::bit(1));
cnode->set_into_collide_mask(CollideMask(0));  // don't let others collide into me
NodePath cnp = player_np.attach_new_node(cnode);

CollisionTraverser ctrav;
PT(CollisionHandlerQueue) queue = new CollisionHandlerQueue();
ctrav.add_collider(cnp, queue);
```

## See also

[CollisionSolid.md](CollisionSolid.md) · [CollisionTraverser.md](CollisionTraverser.md)
· [CollisionEntry.md](CollisionEntry.md) · [README.md](README.md)
