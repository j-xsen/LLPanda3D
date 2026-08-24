# AnimGroup

**Source:** `panda/src/chan/animGroup.h` / `.I` / `.cxx`
**Inherits:** `TypedWritableReferenceCount`, `Namable`
**Inherited by:** [AnimBundle](AnimBundle.md), [AnimChannelBase](AnimChannelBase.md)

Base class for the entire animation-channel hierarchy. An `AnimGroup` tree
models one animation's structure: an [AnimBundle](AnimBundle.md) at the
root, with `AnimChannel` nodes (see [AnimChannelBase](AnimChannelBase.md))
as children/descendants, one per animatable joint or slider. The tree shape
must line up (by name) with the corresponding
[PartGroup](PartGroup.md)/[MovingPartBase](MovingPartBase.md) hierarchy on
the model being animated — [auto_bind](auto_bind.md) and
[PartBundle::bind_anim()](PartBundle.md) are what match the two trees up.

## Behavior notes

- **Never construct directly without a parent.** The public constructor
  requires a non-null `parent` and immediately appends itself to
  `parent->_children`; `nassertv(parent != nullptr)` enforces this. To start
  a hierarchy, create an [AnimBundle](AnimBundle.md) first (its constructor
  sets `_root = this`), then build children off of that.
- **`_root` is inherited down the tree at construction time**, not looked up
  dynamically — each new child copies `parent->_root` into its own `_root`
  member.
- **`sort_descendants()` sorts every level of the hierarchy alphabetically by
  name, recursively.** The header/`.cxx` comment is explicit about why: this
  must be done after building the hierarchy to guarantee that child names
  line up correctly once the bundle is later bound (matched by name) to a
  `PartBundle`.
- **`find_child()` searches the whole subtree; `get_child_named()` searches
  only direct children.** Easy to confuse — reach for `find_child()` unless
  you specifically want one level.
- **`copy_subtree()` produces a deep copy of the group structure but not
  necessarily the leaf data.** [AnimBundle::copy_bundle()](AnimBundle.md)
  documents that channel data tables (e.g. in
  [AnimChannelMatrixXfmTable](AnimChannelMatrixXfmTable.md)) are shared
  between the original and the copy, not duplicated.
- **`get_value_type()` returns `TypeHandle::none()` in the base class** — a
  bit of runtime type-checking so joints and channels can verify they're
  matching by value type (matrix vs. scalar), overridden by concrete channel
  classes.

## API

| Signature | Notes |
|---|---|
| `explicit AnimGroup(AnimGroup *parent, const std::string &name)` | Normal constructor; appends to `parent`'s children |
| `int get_num_children() const` / `AnimGroup *get_child(int n) const` | `MAKE_SEQ`'d as `get_children()` / property `children` |
| `AnimGroup *get_child_named(const std::string &name) const` | Direct children only |
| `AnimGroup *find_child(const std::string &name) const` | Recursive, whole subtree |
| `void sort_descendants()` | Alphabetical sort at every level, recursive; do after building the hierarchy |
| `virtual TypeHandle get_value_type() const` | `TypeHandle::none()` in the base; overridden per value type |
| `virtual void output(std::ostream&) const` / `virtual void write(std::ostream&, int indent_level) const` | One-line vs. full recursive dump |

## Usage

```cpp
#include "animBundle.h"
#include "animChannelMatrixXfmTable.h"

PT(AnimBundle) bundle = new AnimBundle("walk", 24.0, 48);
PT(AnimChannelMatrixXfmTable) hip_channel =
  new AnimChannelMatrixXfmTable(bundle, "hip");

bundle->sort_descendants();

AnimGroup *found = bundle->find_child("hip");
if (found != nullptr) {
  found->write(std::cout, 0);
}
```

## See also

- [AnimBundle](AnimBundle.md) — the required root of any `AnimGroup` tree
- [AnimChannelBase](AnimChannelBase.md) — parent of all concrete channel types
- [PartGroup](PartGroup.md) / [MovingPartBase](MovingPartBase.md) — the
  matching hierarchy on the animated object itself
- [auto_bind](auto_bind.md), [PartBundle](PartBundle.md) — how an `AnimGroup`
  tree gets matched up and bound to a model
- [README.md](README.md)
