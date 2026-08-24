# PartSubset

**Source:** `panda/src/chan/partSubset.h` / `.I` / `.cxx`
**Inherits:** (standalone value type, no base class)

Defines a named subset of joints/sliders to restrict a
[PartBundle::bind_anim()](PartBundle.md) call to — only parts matching the
subset are bound, everything else is left as-is. Passed by value; cheap to
copy (two `pvector<GlobPattern>` lists).

## Behavior notes

- **Include and exclude lists are evaluated independently and consulted
  per-node during the bind walk**, not resolved up front into a fixed part
  list — see [PartGroup::bind_hierarchy()](PartGroup.md): `is_included`
  starts as whatever the parent decided, then gets overridden if this node's
  name matches `matches_include()` or `matches_exclude()`, and that decision
  propagates to children unless they override it again. So a subset can
  include a whole subtree by naming just its root, or carve an exception
  back out further down.
- **An empty include list means "include everything"** —
  `is_include_empty()` is checked by [PartBundle::do_bind_anim()](PartBundle.md)
  to decide whether it can skip the `find_bound_joints()` pre-pass entirely
  and just set `bound_joints = BitArray::all_on()`.
- **Names are `GlobPattern`s**, so `*`/`?` filename-style globbing works,
  e.g. `add_include_joint(GlobPattern("finger_*"))`.
- **`append()`** just concatenates both lists from another `PartSubset` onto
  this one — no de-duplication.

## API

| Signature | Notes |
|---|---|
| `PartSubset()` | Empty subset — matches everything (no restriction) |
| `void add_include_joint(const GlobPattern &name)` | Adds a name/glob to the include list |
| `void add_exclude_joint(const GlobPattern &name)` | Adds a name/glob to the exclude list |
| `void append(const PartSubset &other)` | Concatenates both lists from `other` |
| `bool is_include_empty() const` | True if no include patterns are set (implies "include all") |
| `bool matches_include(const std::string &joint_name) const` | Direct pattern check |
| `bool matches_exclude(const std::string &joint_name) const` | Direct pattern check |
| `void output(std::ostream &out) const` | Also via `operator<<` |

## Usage

```cpp
PartBundle *bundle = new PartBundle("skeleton");
AnimBundle *anim = new AnimBundle("skeleton", 24.0f, 30);

PartSubset upper_body_only;
upper_body_only.add_include_joint(GlobPattern("spine*"));
upper_body_only.add_include_joint(GlobPattern("arm_*"));

PT(AnimControl) control = bundle->bind_anim(anim, 0, upper_body_only);
```

## See also

[PartBundle](PartBundle.md), [PartGroup](PartGroup.md), [README.md](README.md)
