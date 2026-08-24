# MovingPartBase

**Source:** `panda/src/chan/movingPartBase.h` / `.I` / `.cxx`
**Inherits:** [PartGroup](PartGroup.md)
**Inherited by:** `MovingPart<SwitchType>` (see [MovingPartMatrix](MovingPartMatrix.md) / [MovingPartScalar](MovingPartScalar.md))

Base class for a single animatable piece — one joint or slider of a
character — that can be bound to one or more `AnimChannelBase` channels.
`MovingPartBase` has no particular value type; that's added by the templated
`MovingPart<SwitchType>` (documented on [MovingPartMatrix](MovingPartMatrix.md)).

## Behavior notes

- **Channels are stored in a sparse, index-addressed vector**, not a simple
  list: `get_max_bound()` returns one more than the highest bound channel
  index, which may be larger than the actual number of bound channels if
  there are holes (unbound indices). `pick_channel_index()` walks the whole
  part hierarchy to find a reusable hole (or extend past the end) whenever a
  new animation is about to be bound — see [PartBundle::do_bind_anim()](PartBundle.md).
- **Binding to a null `AnimGroup` creates a "default channel"** instead of
  leaving the slot empty: `bind_hierarchy(nullptr, ...)` calls the pure
  virtual `make_default_channel()`, which each `MovingPart<SwitchType>`
  implements as an `AnimChannelFixed<SwitchType>` returning the part's
  original default value. This is how a part not present in a given
  animation still gets a defined (frozen) value rather than undefined
  behavior.
- **Three independent sources can drive a part's value, in priority order:**
  a `_forced_channel` (set by `freeze_joint()`/`control_joint()` via
  `apply_freeze_matrix()`/`apply_control()`) always wins if present;
  otherwise, if exactly one `AnimControl` currently has nonzero blend effect
  on this part, `_effective_channel`/`_effective_control` (computed by
  `determine_effective_channels()`) is used directly; otherwise
  `get_blend_value()` must blend across every channel in
  `PartBundle::CData::_blend`. Only `_num_effective_channels == 1` triggers
  the fast single-channel path — 0 or 2+ channels always fall through to a
  full blend loop in `get_blend_value()` (see [MovingPartMatrix](MovingPartMatrix.md)
  for the actual blend math).
- **`do_update()` short-circuits work when nothing changed:** it only calls
  `get_blend_value()` (recompute the value) when a channel's frame data
  actually changed since last check (`AnimChannelBase::has_changed()` /
  `AnimControl::channel_has_changed()`), and only calls the
  `update_internals()` hook (a subclass extension point, e.g. to push a
  matrix onto a scene-graph node) when either that or the parent's transform
  changed.
- **`pick_channel_index()`'s hole-search mutates the `holes` list it's
  given** — it prunes any hole this part is already using, then appends any
  further holes it discovers past `next`, before recursing into `PartGroup`'s
  version for siblings. Calling code (`PartBundle::do_bind_anim()`) doesn't
  need to know this; it just reads `holes.front()` afterward.

## API

### Channel binding state
| Signature | Notes |
|---|---|
| `int get_max_bound() const` | One more than the highest bound channel index; may exceed the true count if there are holes |
| `AnimChannelBase *get_bound(int n) const` | `nullptr` if index `n` isn't bound; `n` is typically an `AnimControl::get_channel_index()` |
| `virtual TypeHandle get_value_type() const = 0` | Pure virtual; implemented per value type in `MovingPart<SwitchType>` |
| `virtual AnimChannelBase *make_default_channel() const = 0` | Pure virtual; used when binding to a null anim |

### Freeze / control override
| Signature | Notes |
|---|---|
| `virtual bool clear_forced_channel()` | Clears a channel set by `apply_freeze*()`/`apply_control()`; returns whether one was cleared |
| `virtual AnimChannelBase *get_forced_channel() const` | The currently forced channel, or `nullptr` |

### Output
| Signature | Notes |
|---|---|
| `virtual void write(std::ostream&, int) const` | Brief recursive dump, value type + name |
| `virtual void write_with_value(std::ostream&, int) const` | Like `write()`, plus `output_value()` per part |
| `virtual void output_value(std::ostream &out) const = 0` | Pure virtual; per-type value formatting |

### Internal update hooks (called by PartBundle, not user code)
| Signature | Notes |
|---|---|
| `virtual bool do_update(root, root_cdata, parent, parent_changed, anim_changed, current_thread)` | Recursive per-frame update entry point; see Behavior notes |
| `virtual void get_blend_value(const PartBundle *root) = 0` | Pure virtual; recomputes `_value` from bound channels |
| `virtual bool update_internals(root, parent, self_changed, parent_changed, current_thread)` | Hook for subclasses (default: returns `true`, does nothing else) |

## Usage

```cpp
// MovingPartBase is abstract; user code interacts with MovingPartMatrix /
// MovingPartScalar directly. This shows the base-class query surface only,
// using a joint already bound via PartBundle::bind_anim() (see PartBundle.md).
PartBundle *bundle = new PartBundle("skeleton");
MovingPartMatrix *joint = new MovingPartMatrix(bundle, "hips", LMatrix4::ident_mat());

for (int i = 0; i < joint->get_max_bound(); ++i) {
  AnimChannelBase *channel = joint->get_bound(i);
  if (channel != nullptr) {
    // channel index i is in use
  }
}
```

## See also

[MovingPartMatrix](MovingPartMatrix.md), [MovingPartScalar](MovingPartScalar.md),
[PartGroup](PartGroup.md), [PartBundle](PartBundle.md), [PartSubset](PartSubset.md),
[AnimChannelBase.md](AnimChannelBase.md), [AnimControl.md](AnimControl.md),
[README.md](README.md)
