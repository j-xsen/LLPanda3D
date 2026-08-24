# AnimChannelMatrixDynamic

**Source:** `panda/src/chan/animChannelMatrixDynamic.h` / `.I` / `.cxx`
**Inherits:** [AnimChannelMatrix](AnimChannelMatrix.md)

A matrix channel fed from code at runtime instead of from a table baked
into a model file. It operates in one of two mutually-exclusive modes:
**explicit**, where the caller calls `set_value()` every frame to push a
new transform in, or **implicit**, where `set_value_node()` names a
`PandaNode` and the channel reads that node's transform each frame instead.
This is the channel type used to drive a joint procedurally — e.g. IK,
physics ragdolls, or attaching a joint's motion to another part of the
scene graph.

## Behavior notes

- **Setting one mode clears the other.** `set_value()` (either overload)
  calls `_value_node.clear()`; `set_value_node()` overwrites `_value` from
  the node's current transform immediately (so a query right after setting
  it doesn't require waiting for the next frame). Only one of "explicit
  value" or "value node" is ever in effect at a time.
- **In node mode, every `get_*` call re-reads the node's live transform**,
  not a cached snapshot — `has_changed()`, `get_value()`,
  `get_value_no_scale_shear()`, `get_scale()`, `get_hpr()`, `get_quat()`,
  `get_pos()`, and `get_shear()` each independently call
  `_value_node->get_transform()` first when `_value_node` is set. Reading
  several components in the same frame re-queries the node's transform each
  time rather than sharing one read.
- **`has_changed()` ignores its frame-number/fraction arguments entirely**
  — it compares the new `TransformState` pointer against the value cached
  from the previous call and returns whether it differs. The constructor
  seeds `_last_value` to `nullptr`, which is not a value `_value` can ever
  equal, guaranteeing the very first `has_changed()` call reports `true`.
- **All `get_*` accessors are backed by `TransformState`, not raw storage**
  — `get_value()` calls `_value->get_mat()`; `get_scale()`/`get_hpr()`/
  `get_quat()`/`get_pos()`/`get_shear()` each defer to the matching
  `TransformState` accessor. Unlike the base class's default
  implementations (which assert), these all work regardless of mode,
  because a `TransformState` can always decompose itself.
- **`get_value_no_scale_shear()` only recomposes when needed** — if the
  current `TransformState` has scale or shear, it rebuilds the matrix from
  just hpr+pos (via `compose_matrix()`); otherwise it returns `get_mat()`
  unchanged, since there was nothing to strip.

## API

| Signature | Notes |
|---|---|
| `AnimChannelMatrixDynamic(const std::string &name)` | Starts in explicit mode with an identity transform |
| `void set_value(const LMatrix4 &value)` | Explicit mode; clears any value node |
| `void set_value(const TransformState *value)` | Explicit mode, avoids a decompose/recompose round trip |
| `void set_value_node(PandaNode *node)` | Implicit mode; immediately snapshots the node's current transform |
| `const TransformState *get_value_transform() const` | The last explicit value set (stale if in node mode) |
| `PandaNode *get_value_node() const` | The node last passed to `set_value_node()`, or `nullptr` in explicit mode |

Overrides inherited from [AnimChannelMatrix](AnimChannelMatrix.md):
`get_value()`, `get_value_no_scale_shear()`, `get_scale()`, `get_hpr()`,
`get_quat()`, `get_pos()`, `get_shear()`, `has_changed()` — all described
in Behavior notes above.

## Usage

```cpp
#include "animChannelMatrixDynamic.h"
#include "movingPartMatrix.h"
#include "transformState.h"

void drive_joint_explicitly(MovingPartMatrix *joint) {
  PT(AnimChannelMatrixDynamic) channel =
    new AnimChannelMatrixDynamic("procedural");
  joint->add_channel_from_hierarchy(channel);  // conceptual — see MovingPartBase.md for real binding calls

  channel->set_value(TransformState::make_pos(LPoint3(0, 0, 1)));
}

void drive_joint_from_node(AnimChannelMatrixDynamic *channel, PandaNode *driver_node) {
  channel->set_value_node(driver_node);
  // channel now tracks driver_node->get_transform() every frame
}
```

## See also

[AnimChannelMatrix](AnimChannelMatrix.md), [AnimChannelMatrixFixed](AnimChannelMatrixFixed.md),
[AnimChannelMatrixXfmTable](AnimChannelMatrixXfmTable.md),
[AnimChannelScalarDynamic](AnimChannelScalarDynamic.md) (scalar counterpart),
[MovingPartMatrix](MovingPartMatrix.md), [README.md](README.md)
