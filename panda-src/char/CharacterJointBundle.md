# CharacterJointBundle

**Source:** `panda/src/char/characterJointBundle.h` / `.I` / `.cxx`
**Inherits:** [../chan/PartBundle.md](../chan/PartBundle.md)

The collection of all the joints and sliders in a character. Every
[Character](Character.md) constructs one of these for itself; there is
normally no reason to construct a `CharacterJointBundle` directly.

## Behavior notes

- **Normally created only by `Character`'s constructor** (`new
  CharacterJointBundle(name)`), which is also why the copy constructor is
  `protected` — use `make_copy()`/`copy_subgraph()` (inherited from
  [PartGroup](../chan/PartGroup.md)) instead.
- **`add_node()`/`remove_node()` propagate ownership down the whole joint
  hierarchy.** When a `Character` node is associated with this bundle
  (`add_node()`, normally called by `PartBundleNode` machinery, e.g. during
  scene-graph flattening), `r_set_character()` recursively walks every
  descendant and sets `CharacterJoint::set_character()` on each one found.
- **Removing the last associated node does *not* clear the joints' back-
  pointers.** `remove_node()` only calls `r_set_character()` again — pointing
  at the *next* remaining associated node — when `get_num_nodes() > 0` after
  the removal. If the removed node was the only one, the joints are left
  still pointing at the (now-detached) `Character`; the only place that
  actually nulls out a joint's `_character` pointer is
  `Character::r_clear_joint_characters()`, called from `Character`'s
  destructor.
- **`get_node(n)` assumes every associated `PartBundleNode` is a
  `Character`** — it unconditionally `DCAST`s. This holds in practice since
  nothing else constructs a `CharacterJointBundle`, but would misbehave if
  one were manually attached to a plain `PartBundleNode`.

## API

### Construction & copying
| Signature | Notes |
|---|---|
| `explicit CharacterJointBundle(const std::string &name = "")` | Normally not called directly — see Behavior notes |
| `virtual PartGroup *make_copy() const` | Shallow copy, no children, per `PartGroup` convention |

### Node association
| Signature | Notes |
|---|---|
| `INLINE Character *get_node(int n) const` | DCAST wrapper over `PartBundle::get_node()` |
| `virtual void add_node(PartBundleNode *node)` *(protected)* | Propagates the owning `Character` onto every joint — see Behavior notes |
| `virtual void remove_node(PartBundleNode *node)` *(protected)* | Reassigns to the next remaining node, if any — see Behavior notes |

## Usage

```cpp
#include "character.h"
#include "characterJointBundle.h"

PT(Character) actor = new Character("Actor");
CharacterJointBundle *bundle = actor->get_bundle(0);

int num_nodes = bundle->get_num_nodes();  // inherited from PartBundle
for (int i = 0; i < num_nodes; ++i) {
  Character *owner = bundle->get_node(i);
}
```

## See also

[Character](Character.md), [CharacterJoint](CharacterJoint.md),
[CharacterSlider](CharacterSlider.md), [../chan/PartBundle.md](../chan/PartBundle.md),
[../chan/PartGroup.md](../chan/PartGroup.md), [README.md](README.md)
