# AnimChannelBase

**Source:** `panda/src/chan/animChannelBase.h` / `.I` / `.cxx`
**Inherits:** [AnimGroup](AnimGroup.md)
**Inherited by:** `AnimChannel<SwitchType>` (see [AnimChannelMatrix](AnimChannelMatrix.md) / [AnimChannelScalar](AnimChannelScalar.md)), [AnimChannelFixed](AnimChannelFixed.md)

Parent class for every animation channel. An "AnimChannel" (the family of
concrete classes rooted here) is an arbitrary function over frames — usually
a table read from an egg file, but it can be computed or generated any other
way. This class itself carries no value type; that's introduced one level
down by the templated `AnimChannel<SwitchType>` (documented on
[AnimChannelMatrix](AnimChannelMatrix.md)/[AnimChannelScalar](AnimChannelScalar.md))
and by [AnimChannelFixed](AnimChannelFixed.md).

## Behavior notes

- **The no-parent constructor exists only for `AnimChannelFixed`'s benefit.**
  The header comment on the protected `AnimChannelBase(const std::string
  &name)` constructor says explicitly: "don't use this constructor... it
  exists only so that AnimChannelFixed may define itself outside of the
  hierarchy." Every other channel must be constructed with a parent, joining
  the normal `AnimGroup` tree.
- **`has_changed()` always returns `true` in the base class** — a
  conservative default meaning "assume every frame is different." Concrete
  table-driven channels override this to compare frame values and skip
  redundant work when an animation hasn't actually moved between two
  sampled frames (e.g. during a held pose).
- **`get_value_type()` is pure virtual here** — every concrete channel must
  report whether it holds matrix or scalar values, used for runtime
  type-checking against the [MovingPartBase](MovingPartBase.md) it's bound to.

## API

| Signature | Notes |
|---|---|
| `AnimChannelBase(AnimGroup *parent, const std::string &name)` | Normal constructor, joins `parent`'s hierarchy |
| `virtual bool has_changed(int last_frame, double last_frac, int this_frame, double this_frac)` | Base always returns `true`; `last_frac`/`this_frac` are sub-frame blend fractions, nonzero only in frame-blend mode |
| `virtual TypeHandle get_value_type() const = 0` | Pure virtual; matrix vs. scalar |

## Usage

`AnimChannelBase` is never instantiated directly — use a concrete channel
type instead:

```cpp
#include "animBundle.h"
#include "animChannelMatrixXfmTable.h"

PT(AnimBundle) bundle = new AnimBundle("walk", 24.0, 48);
// AnimChannelMatrixXfmTable inherits AnimChannelBase via AnimChannelMatrix.
PT(AnimChannelMatrixXfmTable) channel =
  new AnimChannelMatrixXfmTable(bundle, "hip");

TypeHandle value_type = channel->get_value_type();
```

## See also

- [AnimGroup](AnimGroup.md) — parent class implementing the tree itself
- [AnimChannelMatrix](AnimChannelMatrix.md) / [AnimChannelScalar](AnimChannelScalar.md) — the templated `AnimChannel<SwitchType>` instantiations
- [AnimChannelFixed](AnimChannelFixed.md) — the other direct subclass, for constant-value channels
- [README.md](README.md)
