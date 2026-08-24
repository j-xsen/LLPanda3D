# AnimChannelScalar

**Source:** `panda/src/chan/animChannel.h` / `.I` / `.cxx` (template `AnimChannel<SwitchType>`, instantiated as `AnimChannel<ACScalarSwitchType>`)
**Inherits:** [AnimChannelBase](AnimChannelBase.md)
**Inherited by:** `AnimChannelScalarTable`, `AnimChannelScalarDynamic`, [AnimChannelFixed](AnimChannelFixed.md)`<ACScalarSwitchType>` (all outside this file's scope except the last)

`AnimChannelScalar` is a `typedef` (`animChannel.h`) for
`AnimChannel<ACScalarSwitchType>` — the scalar-valued instantiation of the
same `AnimChannel<SwitchType>` template documented fully in
[AnimChannelMatrix.md](AnimChannelMatrix.md). It drives a single
floating-point value per frame (a morph/blend-shape slider weight, or any
other one-dimensional animatable quantity) rather than a joint transform,
via a `MovingPart<ACScalarSwitchType>` (`MovingPartScalar`). See
AnimChannelMatrix.md for the general template mechanics — `SwitchType` as a
static-member policy class, the protected standalone constructor, why
`init_type()` registers per-instantiation type names, and how
`get_value_type()` is used for runtime binding checks. This page covers only
what differs for the scalar instantiation.

## Behavior notes

- **`ACScalarSwitchType::ValueType` is `PN_stdfloat`**, not `LMatrix4` — so
  `AnimChannelScalar::ValueType` (and every `get_value(int, ValueType&)`
  signature) resolves to a plain float/double rather than a matrix.
- **The five transform-decomposition accessors inherited from
  `AnimChannel<SwitchType>` (`get_scale`, `get_hpr`, `get_quat`, `get_pos`,
  `get_shear`) are still present on `AnimChannelScalar` but are
  categorically meaningless for it** — the header itself says these "only
  have meaning for matrix types," and no scalar leaf subclass overrides
  them. Calling any of the five on any `AnimChannelScalar` (fixed, table, or
  dynamic) always hits the base-class `nassertv(false)`. Unlike
  `AnimChannelMatrix`, there is no subclass carve-out here: for scalar
  channels, `get_value()` and `get_value_no_scale_shear()` (which just
  forwards to `get_value()`, per the base default) are the *only* safe
  accessors, always.
- **`ACScalarSwitchType::output_value()` is a plain stream insertion**
  (`out << value`) — no decomposition step like the matrix switch type's
  scale/shear/hpr/translate breakdown, since there's nothing to decompose.
- **`ACScalarSwitchType::write_datagram()`/`read_datagram()` use
  `Datagram::add_stdfloat()`/`DatagramIterator::get_stdfloat()`** rather
  than delegating to a math-type's own datagram methods (there is no
  `LMatrix4`-style object to delegate to for a bare `PN_stdfloat`).
- **Type names are distinct from the matrix instantiation**:
  `get_channel_type_name()` → `"AnimChannelScalar"`,
  `get_fixed_channel_type_name()` → `"AnimChannelScalarFixed"` (note: not
  the `"AnimChannelFixed<...>"` pattern the matrix switch type uses —
  `AnimChannelFixed<ACScalarSwitchType>` registers under the literal name
  `"AnimChannelScalarFixed"`), `get_part_type_name()` →
  `"MovingPart<PN_stdfloat>"`.

## API

### Value access
| Signature | Notes |
|---|---|
| `typedef PN_stdfloat ValueType` | fixed by `ACScalarSwitchType::ValueType` |
| `virtual void get_value(int frame, PN_stdfloat &value) = 0` | pure virtual; the only accessor guaranteed meaningful |
| `virtual void get_value_no_scale_shear(int frame, PN_stdfloat &value)` | default = `get_value()`; no scalar subclass overrides it |
| `virtual void get_scale/get_hpr/get_quat/get_pos/get_shear(...)` | all `nassertv(false)` always — never meaningful for a scalar channel |
| `virtual TypeHandle get_value_type() const` | returns `TypeHandle` for `PN_stdfloat` |

### Construction / inherited
Same shapes as [AnimChannelMatrix.md](AnimChannelMatrix.md)'s Construction and
"Inherited from AnimChannelBase / AnimGroup" tables, with `ValueType =
PN_stdfloat` substituted throughout.

## Usage

```cpp
#include "animChannel.h"
#include "animChannelFixed.h"

// A standalone fixed scalar channel (e.g. a default morph-slider value with
// no keyframe table needed) — see AnimChannelFixed.md.
PT(AnimChannelFixed<ACScalarSwitchType>) fixed_channel =
  new AnimChannelFixed<ACScalarSwitchType>("blink", 0.0f);

AnimChannelScalar *channel = fixed_channel.p();

PN_stdfloat value;
channel->get_value(0, value);                 // safe
PN_stdfloat value_no_scale_shear;
channel->get_value_no_scale_shear(0, value_no_scale_shear);  // safe: forwards to get_value()

TypeHandle value_type = channel->get_value_type();  // TypeHandle for PN_stdfloat

// channel->get_scale(0, some_vec3) would hit nassertv(false) — always
// meaningless for a scalar channel, never overridden.
```

## See also

[AnimChannelMatrix.md](AnimChannelMatrix.md) (full template mechanics —
`SwitchType`, standalone construction, type registration, binding checks),
[AnimChannelBase.md](AnimChannelBase.md), [AnimChannelFixed.md](AnimChannelFixed.md)
(the constant-value leaf class used in the Usage example above)
