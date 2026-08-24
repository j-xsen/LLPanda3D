# PartGroup

**Source:** `panda/src/chan/partGroup.h` / `.I` / `.cxx`
**Inherits:** `TypedWritableReferenceCount`, `Namable`
**Inherited by:** [PartBundle](PartBundle.md), [MovingPartBase](MovingPartBase.md)

Base class for `PartBundle` and `MovingPart`. Defines a hierarchy of
`MovingPart`s (joints/sliders) — the "part" side of the binding that pairs up
against an `AnimGroup` hierarchy (the "anim" side, see [AnimGroup](AnimGroup.md))
when an animation is bound. A `PartGroup` may only be constructed with a
parent; the root of a hierarchy must be a [PartBundle](PartBundle.md).

## Behavior notes

- **`check_hierarchy()` matches by name, not position**, walking both
  children lists in tandem assuming each is sorted alphabetically (see
  `sort_descendants()`). A part named `"morph"` with no matching anim child
  (or vice versa) is silently ignored as a special case — egg files commonly
  have this asymmetry and it's not treated as an error. Any other mismatch
  fails unless the corresponding `HMF_ok_part_extra`/`HMF_ok_anim_extra` bit
  in `hierarchy_match_flags` is set, in which case `chan` category still logs
  an info-level diff before proceeding.
- **`bind_hierarchy()` also relies on alphabetical order** to walk the part
  and anim children lists in lockstep. A part with no matching anim child
  still gets a recursive `bind_hierarchy(nullptr, ...)` call — see
  [MovingPartBase](MovingPartBase.md) for what binding to a null anim means.
- **`PartSubset` in/exclusion is evaluated per-group during the bind walk**,
  not precomputed: `is_included` toggles based on `subset.matches_include()`/
  `matches_exclude()` at each level and is inherited by children unless
  overridden — see [PartSubset](PartSubset.md).
- **`make_copy()`/`copy_subgraph()` never copy children implicitly** — the
  copy constructor explicitly does not copy `_children`; only
  `copy_subgraph()` walks and duplicates the whole tree, warning via the
  `chan` category if a subclass's `make_copy()` produces the wrong dynamic
  type (a sign a derived class forgot to override `make_copy()`).
- **`apply_freeze()` tries both matrix and scalar forms** — it calls
  `apply_freeze_matrix()` first and, if that returns `false` (not a
  matrix-typed part), falls back to `apply_freeze_scalar()` using the
  transform's X position component as the scalar value. Both base
  implementations here return `false`; only [MovingPartMatrix](MovingPartMatrix.md)
  and [MovingPartScalar](MovingPartScalar.md) actually implement one each.
- **`is_character_joint()`** always returns `false` here; it exists purely so
  `PartGroup::do_update()` and related generic code can cheaply distinguish
  a `CharacterJoint` without a `get_type()` check. Overridden in the `char`
  module.
- Declares `Character`, `CharacterJointBundle`, and `PartBundle` as
  `friend class` — those classes (the first two live in `panda/src/char`)
  reach into `PartGroup` internals that aren't otherwise exposed.

## API

### Hierarchy navigation
| Signature | Notes |
|---|---|
| `PartGroup(PartGroup *parent, const std::string &name)` | The normal constructor; `parent` must not be null |
| `virtual PartGroup *make_copy() const` | Shallow copy, no children; override in every subclass |
| `PartGroup *copy_subgraph() const` | Deep copy of this node and all descendants |
| `int get_num_children() const` / `PartGroup *get_child(int n) const` | Also exposed as the `children` `MAKE_SEQ_PROPERTY` |
| `PartGroup *get_child_named(const std::string &name) const` | Direct children only |
| `PartGroup *find_child(const std::string &name) const` | Recursive search |
| `void sort_descendants()` | Alphabetizes children at every level; required before binding so name-matching in `bind_hierarchy()`/`check_hierarchy()` works |
| `virtual bool is_character_joint() const` | `false` here; `true` in `char`'s `CharacterJoint` |

### Freezing / controlling (delegates to MovingPart subclasses)
| Signature | Notes |
|---|---|
| `bool apply_freeze(const TransformState *transform)` | Tries matrix then scalar; see Behavior notes |
| `virtual bool apply_freeze_matrix(pos, hpr, scale)` | Returns `false` here; see [MovingPartMatrix](MovingPartMatrix.md) |
| `virtual bool apply_freeze_scalar(PN_stdfloat value)` | Returns `false` here; see [MovingPartScalar](MovingPartScalar.md) |
| `virtual bool apply_control(PandaNode *node)` | Returns `false` here |
| `virtual bool clear_forced_channel()` | Undoes a freeze/control; returns `false` here |
| `virtual AnimChannelBase *get_forced_channel() const` | Returns `nullptr` here |

### Output
| Signature | Notes |
|---|---|
| `virtual void write(std::ostream &out, int indent_level) const` | Recursive brief dump |
| `virtual void write_with_value(std::ostream &out, int indent_level) const` | Like `write()`, plus each part's current value |

### Enums
| Value | Meaning |
|---|---|
| `HMF_ok_part_extra` | `check_hierarchy()`/`bind_anim()` tolerate parts with no matching anim node |
| `HMF_ok_anim_extra` | Tolerate anim nodes with no matching part |
| `HMF_ok_wrong_root_name` | Tolerate the bundle name not matching the anim's root name |

## Usage

```cpp
// See PartBundle.md for how a hierarchy is normally built and bound;
// PartGroup itself is rarely constructed or called directly by user code.
PartBundle *bundle = new PartBundle("skeleton");
PartGroup *joint = new PartGroup(bundle, "hips");
bundle->sort_descendants();
```

`vector_PartGroupStar` (`vector_PartGroupStar.h/.cxx`) is just an exported
`pvector<PartGroup*>` instantiation, not a real class — it exists so other
modules linking against `libp3chan` don't each re-instantiate the template.

## See also

[PartBundle](PartBundle.md), [MovingPartBase](MovingPartBase.md),
[MovingPartMatrix](MovingPartMatrix.md), [MovingPartScalar](MovingPartScalar.md),
[PartSubset](PartSubset.md), [AnimGroup](AnimGroup.md), [README.md](README.md)
