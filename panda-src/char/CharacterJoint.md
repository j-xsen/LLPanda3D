# CharacterJoint

**Source:** `panda/src/char/characterJoint.h` / `.I` / `.cxx`
**Inherits:** [../chan/MovingPartMatrix.md](../chan/MovingPartMatrix.md)

One joint of a character's animation: an animating transform matrix, plus
the machinery that composes it with its ancestors into a net (character-
space) transform and a skinning matrix usable by vertex animation.

## Behavior notes

- **`update_internals()` computes `_net_transform` conditionally, then
  propagates it.** If the joint's parent is itself a `CharacterJoint`, the
  net transform is `_value * parent->_net_transform`, recomputed only when
  `parent_changed || self_changed`. If the joint is a top-level joint (its
  parent is the bundle root, not another joint), the net transform is
  `_value * root->get_root_xform()`, recomputed only when `self_changed`.
  When the net transform *does* change: every node registered via
  `add_net_transform()` gets its `PandaNode::set_transform()` updated to
  match, `_skinning_matrix` is recomputed as
  `_initial_net_transform_inverse * _net_transform`, and every dependent
  [JointVertexTransform](JointVertexTransform.md) is told
  `mark_modified()` — this last step is the actual bridge from "a joint
  moved" to "the mesh's soft-skinning is stale." Independently, whenever
  `self_changed` is true, every node registered via `add_local_transform()`
  gets updated with the joint-local `_value` transform, regardless of
  whether the net transform changed.
- **`_initial_net_transform_inverse` is the bind-pose inverse**, computed
  once in the constructor right after the first `update_internals()` call
  establishes the joint's initial `_net_transform`. `do_xform()` (called
  when a transform is baked/flattened onto the joint, e.g. via
  `PartBundle::xform()`) composes the new inverse transform into this
  cached inverse before delegating to `MovingPartMatrix::do_xform()`, so
  the bind pose stays consistent after such a bake.
- **`add_net_transform()`/`add_local_transform()` auto-attach a
  [CharacterJointEffect](CharacterJointEffect.md)** to the target node
  (only if the joint currently has an owning `_character`) — this effect
  forces `Character::update()` on any later query of that node's relative
  transform, so a node "socketed" to a joint always reads a fresh pose even
  if queried before the next cull traversal would have updated it anyway.
  The corresponding `remove_net_transform()`/`remove_local_transform()`
  only clear that effect if it still `matches_character()` this joint's
  character — an effect belonging to a different character (or joint) is
  left alone.
- **"Net" vs "local":** `add_net_transform()` tracks the joint's full
  character-space composed transform; `add_local_transform()` tracks only
  the joint-local `_value`, relative to its immediate parent joint. Neither
  reparents the target node — the caller is responsible for where in the
  scene graph the socketed node actually lives.
- **`_vertex_transforms`** is a private set of
  [JointVertexTransform](JointVertexTransform.md) pointers that register
  themselves with the joint on construction (`JointVertexTransform` is a
  `friend class`) purely so the joint can notify them of skinning-matrix
  changes — a `CharacterJoint` never has to know about `TransformTable`s or
  `Geom`s directly.
- **`set_character()` is `private`** — ownership is managed internally by
  [Character](Character.md) and [CharacterJointBundle](CharacterJointBundle.md)
  (both declared `friend class`), not called directly by user code. Changing
  the owning character also walks every registered net/local transform node,
  swapping their `CharacterJointEffect` to the new character (or clearing it
  if the new character is `nullptr`).
- **Bam compatibility:** `complete_pointers()` only reads back an owning-
  `Character` pointer for bam files with minor version `>= 4`; older files
  leave `_character` `nullptr` after load.

## API

### Construction
| Signature | Notes |
|---|---|
| `explicit CharacterJoint(Character *character, PartBundle *root, PartGroup *parent, const std::string &name, const LMatrix4 &default_value)` | Immediately computes the initial net transform and its inverse |
| `virtual bool is_character_joint() const` | Always `true` — cheap alternative to `is_of_type()` |
| `virtual PartGroup *make_copy() const` | Shallow copy; `_character` resets to `nullptr`, transform-node sets are **not** copied |

### Reading the transform
| Signature | Notes |
|---|---|
| `void get_transform(LMatrix4 &transform) const` / `INLINE const LMatrix4 &get_transform() const` | Joint-local transform (`_value`), relative to the parent joint |
| `CPT(TransformState) get_transform_state() const` | Same, wrapped as a `TransformState` |
| `void get_net_transform(LMatrix4 &transform) const` | Composed transform from the hierarchy root |

### Exposing the joint to scene-graph nodes
| Signature | Notes |
|---|---|
| `bool add_net_transform(PandaNode *node)` | Node tracks the joint's net transform every frame; auto-attaches a [CharacterJointEffect](CharacterJointEffect.md) |
| `bool remove_net_transform(PandaNode *node)` | Also clears the effect if it still matches this joint's character |
| `bool has_net_transform(PandaNode *node) const` | |
| `void clear_net_transforms()` | Removes and un-effects every net-transform node |
| `NodePathCollection get_net_transforms()` | |
| `bool add_local_transform(PandaNode *node)` | Like `add_net_transform()`, but tracks the joint-local `_value` |
| `bool remove_local_transform(PandaNode *node)` / `bool has_local_transform(PandaNode *node) const` | |
| `void clear_local_transforms()` / `NodePathCollection get_local_transforms()` | |

### Ownership
| Signature | Notes |
|---|---|
| `Character *get_character() const` | The owning `Character`, or `nullptr` |

## Usage

```cpp
#include "character.h"
#include "characterJointBundle.h"
#include "characterJoint.h"
#include "pandaNode.h"
#include "nodePath.h"

PT(Character) actor = new Character("Actor");
CharacterJointBundle *bundle = actor->get_bundle(0);
CharacterJoint *hand = actor->find_joint("hand");

if (hand != nullptr) {
  PT(PandaNode) prop_holder = new PandaNode("prop-holder");
  NodePath prop_np(prop_holder);

  hand->add_net_transform(prop_holder);  // stays glued to the hand every frame

  LMatrix4 net;
  hand->get_net_transform(net);

  hand->remove_net_transform(prop_holder);
}
```

## See also

[Character](Character.md), [CharacterJointBundle](CharacterJointBundle.md),
[JointVertexTransform](JointVertexTransform.md), [CharacterJointEffect](CharacterJointEffect.md),
[../chan/MovingPartMatrix.md](../chan/MovingPartMatrix.md),
[../chan/PartGroup.md](../chan/PartGroup.md), [README.md](README.md)
