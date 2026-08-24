# JointVertexTransform

**Source:** `panda/src/char/jointVertexTransform.h` / `.I` / `.cxx`
**Inherits:** [../gobj/VertexTransform.md](../gobj/VertexTransform.md)

A specialization of `VertexTransform` that returns the transform needed to
move a vertex as if it were rigidly assigned to a particular
[CharacterJoint](CharacterJoint.md). This is the fundamental building block
of soft-skinned (and, via a single-entry `TransformTable`, hard-skinned)
character vertex animation: multiple `JointVertexTransform`s with different
weights are combined (via `TransformBlend`) to blend a vertex across several
joints.

The header notes the geometry using this must be parented to the scene
graph at the level of the character's root joint — i.e. **not** under a node
that is itself directly animated by a joint — since the transform returned
here already incorporates the joint's full net (character-space) transform
from the mesh's neutral pose.

## Behavior notes

- **All three matrix queries just read the joint's cached skinning
  matrix.** `get_matrix()`, `mult_matrix()`, and `accumulate_matrix()`
  read/compose/accumulate `_joint->_skinning_matrix` directly — none of them
  do any computation of their own. All the real work
  (`_skinning_matrix = _initial_net_transform_inverse * _net_transform`)
  happens in [CharacterJoint::update_internals()](CharacterJoint.md).
- **Registers itself with the joint on construction** (`_joint
  ->_vertex_transforms.insert(this)`, possible because
  `JointVertexTransform` is a `friend class` of `CharacterJoint`) and
  unregisters on destruction. This is how a joint knows to call
  `mark_modified()` on this object whenever its skinning matrix changes,
  which in turn invalidates any `TransformTable`/`TransformBlendTable` that
  references it (see [VertexTransform](../gobj/VertexTransform.md)'s global
  modification-sequence mechanism) — the joint never needs to know about
  tables or `Geom`s directly.
- **The `.cxx` constructor's doc comment is stale/copy-pasted**: it
  describes "convert[ing] vertices from the indicated joint's coordinate
  space, into the other indicated joint's space," but the constructor only
  takes one joint and does no such two-joint conversion — it simply binds
  to that one joint's skinning matrix.
- **`output()`** prints just the joint's name.
- **The default constructor is `private`**, used only by the bam loader.

## API

| Signature | Notes |
|---|---|
| `JointVertexTransform(CharacterJoint *joint)` | Registers itself with `joint` |
| `INLINE const CharacterJoint *get_joint() const` | |
| `virtual void get_matrix(LMatrix4 &matrix) const` | Copies `joint->_skinning_matrix` |
| `virtual void mult_matrix(LMatrix4 &result, const LMatrix4 &previous) const` | `result = skinning_matrix * previous` |
| `virtual void accumulate_matrix(LMatrix4 &accum, PN_stdfloat weight) const` | `accum += skinning_matrix * weight` |
| `virtual void output(std::ostream &out) const` | Prints the joint's name |

## Usage

```cpp
#include "character.h"
#include "characterJointBundle.h"
#include "characterJoint.h"
#include "jointVertexTransform.h"
#include "transformTable.h"

PT(Character) actor = new Character("Actor");
CharacterJointBundle *bundle = actor->get_bundle(0);
CharacterJoint *hand = actor->find_joint("hand");

if (hand != nullptr) {
  PT(JointVertexTransform) hand_transform = new JointVertexTransform(hand);

  PT(TransformTable) table = new TransformTable;
  size_t index = table->add_transform(hand_transform);

  LMatrix4 skin;
  hand_transform->get_matrix(skin);
}
```

## See also

[CharacterJoint](CharacterJoint.md), [Character](Character.md),
[../gobj/VertexTransform.md](../gobj/VertexTransform.md), [../gobj/TransformTable.md](../gobj/TransformTable.md),
[CharacterVertexSlider](CharacterVertexSlider.md) (the analogous scalar-valued object for morphs),
[README.md](README.md)
