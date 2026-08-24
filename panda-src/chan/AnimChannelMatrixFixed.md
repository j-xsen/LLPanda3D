# AnimChannelMatrixFixed

**Source:** `panda/src/chan/animChannelMatrixFixed.h` / `.I` / `.cxx`
**Inherits:** [AnimChannelMatrix](AnimChannelMatrix.md)

A matrix channel that always returns the same fixed pos/hpr/scale
transform, never changing frame to frame. Despite the name overlap with the
`AnimChannelFixed<SwitchType>` template documented separately, this is a
different, standalone concrete class — not an instantiation of that
template — built directly on `AnimChannel<ACMatrixSwitchType>` with its own
storage for the three component vectors. Useful anywhere a joint needs a
constant offset rather than actual animation (see
[AnimChannelFixed](AnimChannelFixed.md) for the template-based alternative
with the same net effect, usable standalone outside a hierarchy).

## Behavior notes

- **`has_changed()` always returns `false`**, unconditionally — callers
  that skip re-evaluating unchanged channels each frame will correctly
  treat this channel as constant after its first evaluation.
- **Stores components, not a matrix.** `_pos`/`_hpr`/`_scale` are kept
  separately and composed on demand via `compose_matrix()` in `get_value()`
  — there's no cached `LMatrix4`, so every `get_value()` call redoes the
  compose.
- **No shear support** — `get_shear()` always returns `LVecBase3::zero()`;
  there is no `_shear` member. `get_value_no_scale_shear()` composes from
  `_hpr`/`_pos` only with unit scale, matching the base class contract.
- **`output()` extends the base format** by appending
  `: pos <p> hpr <h> scale <s>` after the usual type/name line (via
  `AnimChannelMatrix`'s inherited `output()`), so a `write()`'d hierarchy
  shows the fixed values inline without needing to call the getters
  separately.

## API

| Signature | Notes |
|---|---|
| `AnimChannelMatrixFixed(const std::string &name, const LVecBase3 &pos, const LVecBase3 &hpr, const LVecBase3 &scale)` | Only constructor; values are fixed at construction |
| `virtual void get_value(int frame, LMatrix4 &value)` | Composes `_scale`/`_hpr`/`_pos` into `value`; `frame` is ignored |
| `virtual void get_value_no_scale_shear(int frame, LMatrix4 &value)` | Composes with unit scale, no shear |
| `virtual void get_scale(int frame, LVecBase3 &scale)` | Returns `_scale` |
| `virtual void get_hpr(int frame, LVecBase3 &hpr)` | Returns `_hpr` |
| `virtual void get_quat(int frame, LQuaternion &quat)` | `quat.set_hpr(_hpr)` |
| `virtual void get_pos(int frame, LVecBase3 &pos)` | Returns `_pos` |
| `virtual void get_shear(int frame, LVecBase3 &shear)` | Always `LVecBase3::zero()` |
| `virtual bool has_changed(...)` | Always `false` |
| `virtual void output(std::ostream &out) const` | Appends pos/hpr/scale to the base output |

## Usage

```cpp
#include "animChannelMatrixFixed.h"
#include "luse.h"

PT(AnimChannelMatrixFixed) make_fixed_offset() {
  return new AnimChannelMatrixFixed(
    "fixed_offset",
    LVecBase3(0.0f, 0.0f, 1.0f),   // pos
    LVecBase3(0.0f, 0.0f, 0.0f),   // hpr
    LVecBase3(1.0f, 1.0f, 1.0f));  // scale
}
```

## See also

[AnimChannelMatrix](AnimChannelMatrix.md), [AnimChannelFixed](AnimChannelFixed.md)
(template-based alternative, usable standalone),
[AnimChannelMatrixDynamic](AnimChannelMatrixDynamic.md),
[AnimChannelMatrixXfmTable](AnimChannelMatrixXfmTable.md), [README.md](README.md)
