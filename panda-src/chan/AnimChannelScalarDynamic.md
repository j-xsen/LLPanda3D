# AnimChannelScalarDynamic

**Source:** `panda/src/chan/animChannelScalarDynamic.h` / `.I` / `.cxx`
**Inherits:** [AnimChannelScalar](AnimChannelScalar.md)

A scalar channel fed from code at runtime instead of from a table baked
into a model file — the scalar counterpart to
[AnimChannelMatrixDynamic](AnimChannelMatrixDynamic.md). It operates in one
of two mutually-exclusive modes: **explicit**, where the caller calls
`set_value()` each frame to push a new float in, or **implicit**, where
`set_value_node()` names a `PandaNode` and the channel reads the X
component of that node's position each frame instead. Used to drive a morph
slider or other single-float animated property procedurally.

## Behavior notes

- **Node mode reads only the node's X position, not an arbitrary
  component.** `get_value()` (the `const` inline accessor) returns
  `_value->get_pos()[0]` when a value node is set — there's no way to read
  Y, Z, or any other transform component through this channel; a slider
  driven by a node's transform must encode its value in that node's X
  translate.
- **Unlike `AnimChannelMatrixDynamic`, `get_value()` does not itself
  refresh the cached transform in node mode.** Only `has_changed()` (and
  `set_value_node()`, once, at assignment time) calls
  `_value_node->get_transform()` to update `_value`. `get_value()` just
  reads whatever `_value` was cached last. This works correctly under the
  normal calling convention — a channel binding always calls
  `has_changed()` once per frame before `get_value()` — but calling
  `get_value()` in isolation, without a preceding `has_changed()` this
  frame, can return a stale value if the driver node moved since the last
  `has_changed()` call.
- **Explicit mode's `has_changed()` doesn't compare old vs. new value at
  all** — it just returns a sticky `_value_changed` flag that `set_value()`
  sets `true` and `has_changed()` clears back to `false` after reading. This
  means calling `set_value()` with the *same* number it already had still
  reports a change on the next `has_changed()` call, unlike node mode (which
  does a real `TransformState` pointer inequality check).
- **Switching from explicit to node mode synthesizes a fake "previous"
  transform so the switch itself registers as a change.** The first
  `set_value_node()` call after being in explicit mode sets `_last_value` to
  `TransformState::make_pos(_float_value, 0, 0)` before assigning
  `_value_node` — ensuring the very next `has_changed()` correctly detects a
  change if the node's initial X differs from the old explicit float value
  (or even if it happens to match, since a `TransformState` pointer
  comparison, not a numeric one, decides `has_changed()` in node mode —
  practically speaking the switch itself typically registers as a change
  because a freshly-queried transform is a different pointer than the
  synthesized one).
- **The constructor seeds both explicit and node state**, even though only
  one is active at a time: `_float_value = 0`, `_value = _last_value =
  TransformState::make_identity()`, `_value_changed = true` — guaranteeing
  the first `has_changed()` call (in whichever mode is used first) reports
  `true`.

## API

| Signature | Notes |
|---|---|
| `AnimChannelScalarDynamic(const std::string &name)` | Starts in explicit mode, value `0.0` |
| `void set_value(PN_stdfloat value)` | Explicit mode; clears any value node |
| `void set_value_node(PandaNode *node)` | Implicit mode; reads the node's X position each frame |
| `PN_stdfloat get_value() const` | Current value in either mode; see Behavior notes on staleness |
| `PandaNode *get_value_node() const` | The node last passed to `set_value_node()`, or `nullptr` in explicit mode |
| `virtual void get_value(int frame, PN_stdfloat &value)` | Override; delegates to `get_value() const`, ignores `frame` |
| `virtual bool has_changed(int, double, int, double)` | See Behavior notes — semantics differ by mode |

## Usage

```cpp
#include "animChannelScalarDynamic.h"
#include "movingPartScalar.h"

void drive_slider_explicitly(AnimChannelScalarDynamic *channel, PN_stdfloat blink_amount) {
  channel->set_value(blink_amount);  // e.g. 0.0 = eyes open, 1.0 = closed
}

void drive_slider_from_node(AnimChannelScalarDynamic *channel, PandaNode *driver_node) {
  channel->set_value_node(driver_node);
  // channel now tracks driver_node->get_transform()'s X position every frame,
  // as long as has_changed() is polled once per frame by the normal binding.
}

void read_current_value(AnimChannelScalarDynamic *channel) {
  PN_stdfloat value = channel->get_value();
}
```

## See also

[AnimChannelScalar](AnimChannelScalar.md), [AnimChannelFixed](AnimChannelFixed.md),
[AnimChannelScalarTable](AnimChannelScalarTable.md),
[AnimChannelMatrixDynamic](AnimChannelMatrixDynamic.md) (matrix counterpart),
[MovingPartScalar](MovingPartScalar.md), [README.md](README.md)
