# CharacterSlider

**Source:** `panda/src/char/characterSlider.h` / `.cxx` (no `.I`)
**Inherits:** [../chan/MovingPartScalar.md](../chan/MovingPartScalar.md)

A morph slider within a character: a single floating-point value, typically
animating between 0 and 1, controlling the effect of one or more morphs
(blend shapes). The slider's own value comes entirely from the inherited
`MovingPartScalar`/`MovingPartBase` machinery (bound animation channels,
`freeze_joint()`, `control_joint()`, etc. via the owning
[../chan/PartBundle.md](../chan/PartBundle.md)) — this class's only job is
notifying the [CharacterVertexSlider](CharacterVertexSlider.md) objects that
expose that value to vertex animation.

## Behavior notes

- **`update_internals()` always reports changed.** Unlike
  [CharacterJoint](CharacterJoint.md)'s conditional net-transform
  recomputation, `CharacterSlider::update_internals()` ignores all four of
  its parameters (`root`, `parent`, `self_changed`, `parent_changed`) and
  unconditionally calls `mark_modified()` on every registered
  [CharacterVertexSlider](CharacterVertexSlider.md) in `_vertex_sliders`,
  then returns `true`. Every `update()` pass touches every dependent vertex
  slider observer, even on a frame where the slider's value didn't actually
  move.
- **`_vertex_sliders` is populated by `CharacterVertexSlider` itself**
  (`friend class CharacterVertexSlider`) — a `CharacterVertexSlider`
  inserts itself on construction and erases itself on destruction; a
  `CharacterSlider` never directly constructs or owns its observers.
- **The destructor asserts `_vertex_sliders.empty()`** — every
  `CharacterVertexSlider` referencing this slider must be destroyed first.

## API

### Construction
| Signature | Notes |
|---|---|
| `explicit CharacterSlider(PartGroup *parent, const std::string &name)` | Default value `0` |
| `explicit CharacterSlider(PartGroup *parent, const std::string &name, const PN_stdfloat &default_value)` | |
| `virtual PartGroup *make_copy() const` | Shallow copy; `_vertex_sliders` observers are **not** copied |

### Update
| Signature | Notes |
|---|---|
| `virtual bool update_internals(PartBundle *root, PartGroup *parent, bool self_changed, bool parent_changed, Thread *current_thread)` | Always returns `true`; unconditionally notifies observers — see Behavior notes |

## Usage

```cpp
#include "character.h"
#include "characterJointBundle.h"
#include "characterSlider.h"

PT(Character) actor = new Character("Actor");
CharacterJointBundle *bundle = actor->get_bundle(0);

CharacterSlider *smile = new CharacterSlider(bundle, "smile", 0.0f);

// Freeze/release via the owning bundle (inherited PartBundle API):
bundle->freeze_joint("smile", 0.75f);
bundle->release_joint("smile");

PN_stdfloat current = smile->get_value();  // inherited from MovingPartScalar
```

## See also

[Character](Character.md), [CharacterVertexSlider](CharacterVertexSlider.md),
[../chan/MovingPartScalar.md](../chan/MovingPartScalar.md),
[../chan/PartBundle.md](../chan/PartBundle.md), [README.md](README.md)
