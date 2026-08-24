# MovingPartScalar

**Source:** `panda/src/chan/movingPartScalar.h` / `.I` / `.cxx`
**Inherits:** `MovingPart<ACScalarSwitchType>` → [MovingPartBase](MovingPartBase.md)

The concrete `MovingPart` specialization for scalar-valued parts (`PN_stdfloat`)
— used for morph sliders. Same `MovingPart<SwitchType>` template as
[MovingPartMatrix](MovingPartMatrix.md); see that page for the shared
template mechanics (`_value`/`_default_value`, the forced-channel/
effective-channel priority scheme, `make_default_channel()`). This page
covers only what differs for the scalar type.

## Behavior notes

- **`get_blend_value()`'s blend math is a single weighted average**, not the
  four-mode `BlendType`-driven system `MovingPartMatrix` uses — there's no
  meaningful "componentwise" or quaternion distinction for a single float.
  It still respects `_frame_blend_flag` (blends between an animation's
  current and next frame by `AnimControl::get_frac()`) and still falls back
  to `_default_value` on zero net effect only when `restore-initial-pose` is
  set.
- **`apply_freeze_scalar()`** installs an `AnimChannelFixed<ACScalarSwitchType>`
  directly (there's no separate `AnimChannelScalarFixed` class the way
  matrix has `AnimChannelMatrixFixed` — the generic template is used as-is).
  **`apply_control()`** installs an
  [AnimChannelScalarDynamic](AnimChannelScalarDynamic.md) reading the given
  node's X transform component each frame.
- `MovingPartScalar` has no `apply_freeze_matrix()` override (inherits
  `PartGroup`'s `false`-returning base), so `apply_freeze()`
  ([PartGroup.md](PartGroup.md)) always falls through to
  `apply_freeze_scalar()` for a scalar part.

## API

### Construction
| Signature | Notes |
|---|---|
| `MovingPartScalar(PartGroup *parent, const std::string &name, const PN_stdfloat &default_value = 0)` | Normal constructor |
| `ValueType get_value() const` / `get_default_value() const` | Inherited from `MovingPart<SwitchType>`, `ValueType == PN_stdfloat` |

### Blend / freeze / control
| Signature | Notes |
|---|---|
| `virtual void get_blend_value(const PartBundle *root)` | Single weighted-average blend; see Behavior notes |
| `virtual bool apply_freeze_scalar(PN_stdfloat value)` | Always returns `true` |
| `virtual bool apply_control(PandaNode *node)` | Always returns `true` |

## Usage

```cpp
PartBundle *bundle = new PartBundle("face");
MovingPartScalar *smile = new MovingPartScalar(bundle, "smile", 0.0f);

bundle->freeze_joint("smile", 0.75f);   // hold the slider at 0.75
bundle->release_joint("smile");         // hand control back to bound animations

PN_stdfloat current = smile->get_value();
```

## See also

[MovingPartMatrix](MovingPartMatrix.md) (full template mechanics),
[MovingPartBase](MovingPartBase.md),
[AnimChannelScalarDynamic](AnimChannelScalarDynamic.md),
[AnimChannelScalarTable](AnimChannelScalarTable.md),
[PartBundle](PartBundle.md),
[../char/CharacterSlider.md](../char/CharacterSlider.md) (the `char`
module's concrete use of this class), [README.md](README.md)
