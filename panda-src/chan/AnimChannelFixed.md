# AnimChannelFixed

**Source:** `panda/src/chan/animChannelFixed.h` / `.I` / `.cxx` (template `AnimChannelFixed<SwitchType>`, instantiated as `AnimChannelFixed<ACMatrixSwitchType>` and `AnimChannelFixed<ACScalarSwitchType>`)
**Inherits:** `AnimChannel<SwitchType>` — i.e. [AnimChannelMatrix](AnimChannelMatrix.md) for the matrix instantiation, [AnimChannelScalar](AnimChannelScalar.md) for the scalar instantiation
**Inherited by:** (no further subclasses)

`AnimChannelFixed<SwitchType>` is a special-case `AnimChannel` that returns
the exact same constant value on every single frame, forever. Per the header
comment, it's "a special channel, in that it need not be assigned within a
hierarchy" — unlike every other `AnimChannel`, which must be built as a node
of an `AnimBundle`-rooted tree, an `AnimChannelFixed` can be constructed
standalone with no parent at all. It exists so that parts of the system
needing *some* channel to bind against — most notably a joint
(`MovingPart<SwitchType>`) that has no real animation data for a given
`AnimBundle` — can be handed a trivial "do nothing, just hold this value"
channel created on the fly, rather than requiring every joint to always have
a fully authored table-driven channel. Both instantiations are covered on
this one page since the only difference between them is `ValueType`
(`LMatrix4` for the matrix form, `PN_stdfloat` for the scalar form) —
see [AnimChannelMatrix.md](AnimChannelMatrix.md) and
[AnimChannelScalar.md](AnimChannelScalar.md) for what `SwitchType` supplies.

## Behavior notes

- **No hierarchy required — this is the whole point of the class.** The
  public constructor `AnimChannelFixed(const std::string &name, const
  ValueType &value)` calls `AnimChannel<SwitchType>`'s *protected*,
  no-parent constructor (the one whose header comment says "don't use this
  constructor... it exists only so that AnimChannelFixed may define itself
  outside of the hierarchy"). Every other concrete `AnimChannel` subclass
  must instead go through the public `AnimChannel(AnimGroup *parent, const
  std::string &name)` constructor and be attached beneath an `AnimBundle`.
- **`get_value(int frame, ValueType &value)` ignores `frame` entirely** —
  it just copies the stored `_value` out, regardless of what frame number is
  requested. There is no keyframe table, no interpolation, and no bounds
  checking on `frame` because the frame number is never even looked at.
- **`has_changed()` always returns `false`**, overriding the
  `AnimChannelBase` default (which always returns `true`). Signature is
  `has_changed(int last_frame, double last_frac, int this_frame, double
  this_frac)` and every parameter is ignored — a fixed channel's value can
  never change frame-to-frame, so callers that use `has_changed()` as a
  cheap "do I need to recompute anything" check will correctly skip work for
  a channel bound to a fixed default.
- **Does not override `get_value_no_scale_shear()`, `get_scale()`,
  `get_hpr()`, `get_quat()`, `get_pos()`, or `get_shear()`.** These all fall
  through to the `AnimChannel<SwitchType>` base defaults — meaning even for
  the *matrix* instantiation, `get_scale()`/`get_hpr()`/`get_quat()`/
  `get_pos()`/`get_shear()` hit `nassertv(false)` on an
  `AnimChannelFixed<ACMatrixSwitchType>`, exactly as they would on any other
  un-overriding `AnimChannelMatrix` subclass. `get_value_no_scale_shear()`
  falls back to plain `get_value()` (the base default), which happens to be
  correct here since the one fixed value has no separate "with scale/shear"
  vs. "without" distinction to make. See
  [AnimChannelMatrix.md](AnimChannelMatrix.md)'s Behavior notes for the full
  explanation of why those accessors are unsafe in general.
- **`output()` appends the value to the base `AnimGroup::output()`** —
  `AnimChannel<SwitchType>::output()` is not itself overridden anywhere in
  this template chain (it's inherited straight from `AnimGroup`), so
  `AnimChannelFixed::output()` calls `AnimChannel<SwitchType>::output(out)`
  first (which ultimately runs `AnimGroup::output()`, printing the node's
  name and type) and then appends `" = " << _value` — for the matrix
  instantiation this prints the raw `LMatrix4` via its own `operator<<`
  (not `ACMatrixSwitchType::output_value()`'s decomposed form — that
  helper is used elsewhere, not by this `output()`), for the scalar
  instantiation it prints the bare float.
- **The copy constructor (`AnimChannelFixed(AnimGroup *parent, const
  AnimChannelFixed<SwitchType> &copy)`) is `protected` and, unlike the
  primary constructor, does take a `parent`** — it exists only for
  `make_copy()`-style duplication when an `AnimChannelFixed` happens to be
  copied as part of duplicating a larger hierarchy that contains it (even
  though it need not live in one), mirroring the copy-constructor pattern
  every other `AnimChannel` subclass follows.
- **Registers under a per-`SwitchType` type name, not a shared
  `"AnimChannelFixed"` name**: `SwitchType::get_fixed_channel_type_name()`
  gives `"AnimChannelFixed<LMatrix4>"` for the matrix instantiation but the
  differently-patterned literal `"AnimChannelScalarFixed"` for the scalar
  instantiation (see [AnimChannelScalar.md](AnimChannelScalar.md)'s
  Behavior notes) — there is no single registered `TypeHandle` covering
  both.
- **No Bam `write_datagram`/`fillin` override** — neither `.h` nor `.cxx`
  defines any; a fixed channel is apparently not expected to round-trip its
  `_value` through Bam I/O on this class alone (contrast with
  `ACMatrixSwitchType`/`ACScalarSwitchType`'s `write_datagram`/
  `read_datagram` static helpers, which exist for use by other, table-based
  subclasses).

## API

| Signature | Notes |
|---|---|
| `AnimChannelFixed(const std::string &name, const ValueType &value)` | public; the normal, no-parent-required constructor |
| `AnimChannelFixed(AnimGroup *parent, const AnimChannelFixed<SwitchType> &copy)` | protected; used by `make_copy()`-style duplication only |
| `virtual bool has_changed(int last_frame, double last_frac, int this_frame, double this_frac)` | always returns `false` |
| `virtual void get_value(int frame, ValueType &value)` | ignores `frame`; copies out `_value` |
| `virtual void output(std::ostream &out) const` | base `AnimGroup`/`AnimChannel` output, then `" = " << _value` |
| `ValueType _value` | public data member holding the constant value |

Everything else (`get_value_no_scale_shear`, `get_scale`, `get_hpr`,
`get_quat`, `get_pos`, `get_shear`, `get_value_type`) is inherited unchanged
from `AnimChannel<SwitchType>` — see the matrix/scalar pages' API tables for
exact signatures and which ones are safe to call.

## Usage

```cpp
#include "animChannelFixed.h"
#include "luse.h"

// Matrix instantiation: a fixed identity-transform channel for a joint that
// has no real animation data in this particular AnimBundle.
LMatrix4 identity_xform = LMatrix4::ident_mat();
PT(AnimChannelFixed<ACMatrixSwitchType>) fixed_matrix_channel =
  new AnimChannelFixed<ACMatrixSwitchType>("default_joint", identity_xform);

LMatrix4 matrix_value;
fixed_matrix_channel->get_value(0, matrix_value);     // frame ignored, always identity_xform

bool matrix_changed = fixed_matrix_channel->has_changed(0, 0.0, 1, 0.0);  // always false

// Scalar instantiation: a fixed slider default.
PT(AnimChannelFixed<ACScalarSwitchType>) fixed_scalar_channel =
  new AnimChannelFixed<ACScalarSwitchType>("default_morph", 0.0f);

PN_stdfloat scalar_value;
fixed_scalar_channel->get_value(0, scalar_value);     // frame ignored, always 0.0f

bool scalar_changed = fixed_scalar_channel->has_changed(0, 0.0, 1, 0.0);  // always false
```

## See also

[AnimChannelMatrix.md](AnimChannelMatrix.md) (full template mechanics for
the matrix instantiation, including why `get_scale`/`get_hpr`/etc. remain
unsafe here), [AnimChannelScalar.md](AnimChannelScalar.md) (scalar
instantiation specifics), [AnimChannelBase.md](AnimChannelBase.md)
