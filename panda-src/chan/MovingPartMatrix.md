# MovingPartMatrix

**Source:** `panda/src/chan/movingPartMatrix.h` / `.I` / `.cxx` (template mechanics in `movingPart.h` / `.I`)
**Inherits:** `MovingPart<ACMatrixSwitchType>` → [MovingPartBase](MovingPartBase.md)

The concrete `MovingPart` specialization for matrix-valued parts — this is
what a character joint actually is. `MovingPart<SwitchType>` is a template
(`typedef MovingPart<ACMatrixSwitchType> MovingPartMatrix` via
`EXPORT_TEMPLATE_CLASS`); this page documents the template's full mechanics
using this instantiation, since there is no separate template doc page (see
[MovingPartScalar.md](MovingPartScalar.md) for the scalar instantiation,
which shares everything described here except the value type and blend
math).

## Behavior notes

- **The `MovingPart<SwitchType>` template adds exactly two members over
  `MovingPartBase`:** `_value` (the current computed `ValueType`, `LMatrix4`
  here) and `_default_value` (what to fall back to when nothing drives the
  part). `ValueType` and `ChannelType` (`AnimChannel<SwitchType>`) are
  derived from the `SwitchType` template parameter via
  `typename SwitchType::ValueType`.
- **`make_default_channel()`** decomposes `_default_value` into
  pos/hpr/scale/shear and constructs an
  [AnimChannelMatrixFixed](AnimChannelMatrixFixed.md) from the components
  (shear is dropped) — used when a joint isn't present in a bound
  animation, per [MovingPartBase](MovingPartBase.md)'s default-channel
  mechanism.
- **`get_blend_value()` implements four distinct blend algorithms**, selected
  by `PartBundle::CData::_blend_type` (see [PartBundle](PartBundle.md)'s
  `BlendType` enum) — this is the actual per-frame math that determines how
  a character's pose is computed:
  - `BT_linear` — componentwise-averages the raw matrices. Cheap, but can
    stretch/squash limbs since scale/shear blend along with
    position/rotation.
  - `BT_normalized_linear` (the engine default) — blends position/rotation
    without scale/shear, then blends and reapplies scale/shear separately,
    avoiding the stretch artifacts of plain linear blending. Requires a
    fully-connected skeleton or parts can visually detach.
  - `BT_componentwise` — decomposes every channel into scale/hpr/pos/shear
    and blends each component independently, then recomposes.
  - `BT_componentwise_quat` — same as componentwise, but rotation blends as
    a quaternion instead of HPR (avoids gimbal/interpolation artifacts on
    rotation specifically).
  All four modes also handle inter-frame blending (`_frame_blend_flag`)
  identically within their own math: when set, each animation's current and
  next frame are both sampled and mixed by `AnimControl::get_frac()` before
  being combined across animations.
- **A forced channel always uses frame 0** — `get_blend_value()`'s very first
  check is `_forced_channel != nullptr`, which reads `channel->get_value(0,
  _value)` unconditionally, skipping all blending. This is what
  `apply_freeze_matrix()`/`apply_control()` rely on (they install a
  single-frame or dynamic channel as `_forced_channel`).
- **If total blend effect is zero** (`net_effect == 0.0f`) after summing all
  contributing channels' weights, `_value` is only reset to
  `_default_value` when the `restore-initial-pose` config variable is true
  — otherwise it's left unchanged (holds the last computed pose). See
  [README.md](README.md) config table.
- **`apply_freeze_matrix()`** installs a fresh
  [AnimChannelMatrixFixed](AnimChannelMatrixFixed.md) as `_forced_channel`;
  **`apply_control()`** installs an
  [AnimChannelMatrixDynamic](AnimChannelMatrixDynamic.md) wired to read the
  given `PandaNode`'s transform every frame via `set_value_node()`. Both are
  called only via [PartGroup::apply_freeze()](PartGroup.md) /
  `apply_control()` from [PartBundle::freeze_joint()](PartBundle.md) /
  `control_joint()`.

## API

### Construction
| Signature | Notes |
|---|---|
| `MovingPartMatrix(PartGroup *parent, const std::string &name, const LMatrix4 &default_value)` | Normal constructor |
| `ValueType get_value() const` | Current computed matrix (from `MovingPart<SwitchType>`) |
| `ValueType get_default_value() const` | The value used when nothing drives this part |

### Blend / freeze / control (overrides of MovingPartBase/PartGroup)
| Signature | Notes |
|---|---|
| `virtual AnimChannelBase *make_default_channel() const` | See Behavior notes |
| `virtual void get_blend_value(const PartBundle *root)` | The 4-mode blend implementation described above |
| `virtual bool apply_freeze_matrix(const LVecBase3 &pos, const LVecBase3 &hpr, const LVecBase3 &scale)` | Always returns `true` — this is the class that makes matrix freezing possible |
| `virtual bool apply_control(PandaNode *node)` | Always returns `true` |

## Usage

```cpp
PartBundle *bundle = new PartBundle("skeleton");
MovingPartMatrix *hips = new MovingPartMatrix(bundle, "hips", LMatrix4::ident_mat());

// Freeze the joint to a fixed pose, bypassing any bound animation:
bundle->freeze_joint("hips", LVecBase3(0, 0, 0), LVecBase3(0, 0, 0), LVecBase3(1, 1, 1));

// Or drive it from another node's transform instead:
PT(PandaNode) driver = new PandaNode("hip-driver");
bundle->control_joint("hips", driver);

// Undo either of the above:
bundle->release_joint("hips");

LMatrix4 current = hips->get_value();
```

## See also

[MovingPartScalar](MovingPartScalar.md), [MovingPartBase](MovingPartBase.md),
[AnimChannelMatrixFixed](AnimChannelMatrixFixed.md),
[AnimChannelMatrixDynamic](AnimChannelMatrixDynamic.md),
[AnimChannelMatrixXfmTable](AnimChannelMatrixXfmTable.md),
[PartBundle](PartBundle.md), [../char/CharacterJoint.md](../char/CharacterJoint.md)
(the `char` module's concrete use of this class), [README.md](README.md)
