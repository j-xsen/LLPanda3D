# AnimChannelMatrix

**Source:** `panda/src/chan/animChannel.h` / `.I` / `.cxx` (template `AnimChannel<SwitchType>`, instantiated as `AnimChannel<ACMatrixSwitchType>`)
**Inherits:** [AnimChannelBase](AnimChannelBase.md)
**Inherited by:** `AnimChannelMatrixXfmTable`, `AnimChannelMatrixDynamic`, [AnimChannelFixed](AnimChannelFixed.md)`<ACMatrixSwitchType>` (all outside this file's scope except the last)

`AnimChannelMatrix` is a `typedef` (`animChannel.h`) for `AnimChannel<ACMatrixSwitchType>` —
the matrix-valued instantiation of the `AnimChannel<SwitchType>` template. It is the
abstract base for every animation channel that drives a joint's full transform
(scale/shear/rotation/translation collapsed into one `LMatrix4` per frame), as
opposed to a single scalar slider value. It is never instantiated directly —
only through a concrete leaf subclass (an `.egg`-table-backed
`AnimChannelMatrixXfmTable`, a procedural `AnimChannelMatrixDynamic`, or the
constant-value `AnimChannelFixed<ACMatrixSwitchType>`) attached beneath an
`AnimBundle` hierarchy, then bound to a `MovingPart<ACMatrixSwitchType>`
(`MovingPartMatrix`) joint via a `PartBundle`/`AnimControl`.

## Behavior notes

- **`SwitchType` is a compile-time policy class, not a runtime object.**
  `ACMatrixSwitchType` (defined in `animChannel.h`) supplies everything the
  template needs to specialize itself for matrices, entirely via `static`
  members — there is never an instance of `ACMatrixSwitchType` anywhere:
  - `typedef LMatrix4 ValueType;` — fixes `AnimChannel<ACMatrixSwitchType>::ValueType` to `LMatrix4`.
  - `get_channel_type_name()` → `"AnimChannelMatrix"`, used by `init_type()` to register the `TypeHandle` under that name (not `"AnimChannel"`).
  - `get_fixed_channel_type_name()` → `"AnimChannelFixed<LMatrix4>"`, the type name `AnimChannelFixed<ACMatrixSwitchType>` registers under.
  - `get_part_type_name()` → `"MovingPart<LMatrix4>"`, used by the corresponding `MovingPart<ACMatrixSwitchType>` (`MovingPartMatrix`).
  - `output_value()` — pretty-prints an `LMatrix4` by decomposing it (via `compose_matrix.h`'s `decompose_matrix()`) into scale/shear/hpr/translate and only printing the components that differ from identity/zero, e.g. `" scale 2 hpr (90, 0, 0) trans (1, 2, 3)"`; falls back to printing the raw matrix (`" mat ..."`) if decomposition fails (e.g. a non-affine matrix).
  - `write_datagram()`/`read_datagram()` — delegate to `LMatrix4::write_datagram()`/`read_datagram()` for Bam I/O.
- **`get_value()` is pure virtual here** (`= 0`) — `AnimChannelMatrix` itself
  supplies no per-frame data; every leaf subclass must implement how frame
  number maps to `LMatrix4`.
- **The scale/shear/rotation/translation decomposition accessors
  (`get_value_no_scale_shear()`, `get_scale()`, `get_hpr()`, `get_quat()`,
  `get_pos()`, `get_shear()`) are stubs at this level, not real
  decomposition logic:**
  - `get_value_no_scale_shear()` defaults to just calling `get_value()` —
    i.e. "no scale/shear to strip" unless a subclass overrides it.
  - `get_scale()`, `get_hpr()`, `get_quat()`, `get_pos()`, `get_shear()` all
    default to `nassertv(false)` — an assertion failure (or silent no-op in
    an NDEBUG release build) if called on a channel that doesn't override
    them. These only have real implementations on subclasses that store
    the transform pre-decomposed into separate components (notably
    `AnimChannelMatrixXfmTable`, which keeps per-component frame tables and
    can answer each of these without recomposing the whole matrix). A
    caller that only knows it has an `AnimChannelMatrix*` cannot assume any
    of these five will work — `get_value()` is the only universally safe
    accessor.
  - `AnimChannelFixed<ACMatrixSwitchType>` — the one matrix-valued leaf
    class documented alongside this template family — does **not**
    override any of these five, so calling `get_scale()`/`get_hpr()`/etc.
    on a fixed matrix channel hits the base-class assert; only
    `get_value()` (and `get_value_no_scale_shear()`, which falls back to
    `get_value()`) are safe to call on it. See
    [AnimChannelFixed.md](AnimChannelFixed.md).
- **`get_value_type()` returns `get_type_handle(ValueType)`**, i.e. the
  `TypeHandle` for `LMatrix4` — this is how `MovingPartMatrix::bind_to()`
  (elsewhere in `panda/src/chan`) can runtime-check that a channel and the
  joint it's binding to actually agree on value type before wiring them
  together, since both matrix and scalar channels share the same abstract
  `AnimChannelBase` base with no compile-time type link between a joint and
  its channel.
- **Both constructors that take just a name (no parent) are `protected`.**
  Per the header comment, an `AnimChannel` normally must be built as part of
  an `AnimBundle`-rooted hierarchy (`AnimChannel(AnimGroup *parent, const
  std::string &name)` is the public one). The no-parent constructor exists
  solely so `AnimChannelFixed` can stand alone outside any hierarchy (see
  [AnimChannelFixed.md](AnimChannelFixed.md)).
- **`init_type()` registers under `SwitchType::get_channel_type_name()`**
  (`"AnimChannelMatrix"`), not a literal `"AnimChannel"` — the template
  mechanism means there is effectively no `TypeHandle` for the untyped
  `AnimChannel<SwitchType>` name itself; each instantiation gets its own
  distinct registered type name.
- **No Bam `write_datagram`/`fillin` at this level** — the header comment
  notes this class is abstract and defines no new persistent data of its
  own (`ACMatrixSwitchType::write_datagram`/`read_datagram` exist only to be
  called by leaf subclasses that do store `LMatrix4` frame data).

## API

### Construction (protected/internal — see Behavior notes)
| Signature | Notes |
|---|---|
| `AnimChannel(const std::string &name = "")` | protected; standalone use only (`AnimChannelFixed`) |
| `AnimChannel(AnimGroup *parent, const AnimChannel &copy)` | protected; used by `make_copy()` in subclasses |
| `AnimChannel(AnimGroup *parent, const std::string &name)` | public; normal hierarchy-attached construction |

### Value access
| Signature | Notes |
|---|---|
| `typedef LMatrix4 ValueType` | fixed by `ACMatrixSwitchType::ValueType` |
| `virtual void get_value(int frame, LMatrix4 &value) = 0` | pure virtual; only universally safe accessor |
| `virtual void get_value_no_scale_shear(int frame, LMatrix4 &value)` | default = `get_value()`; meaningful override only on subclasses that store decomposed components |
| `virtual void get_scale(int frame, LVecBase3 &scale)` | default `nassertv(false)` — unsafe unless overridden |
| `virtual void get_hpr(int frame, LVecBase3 &hpr)` | default `nassertv(false)` — unsafe unless overridden |
| `virtual void get_quat(int frame, LQuaternion &quat)` | default `nassertv(false)` — unsafe unless overridden |
| `virtual void get_pos(int frame, LVecBase3 &pos)` | default `nassertv(false)` — unsafe unless overridden |
| `virtual void get_shear(int frame, LVecBase3 &shear)` | default `nassertv(false)` — unsafe unless overridden |
| `virtual TypeHandle get_value_type() const` | returns `TypeHandle` for `LMatrix4` |

### Inherited from AnimChannelBase / AnimGroup
| Signature | Notes |
|---|---|
| `virtual bool has_changed(int last_frame, double last_frac, int this_frame, double this_frac)` | base default returns `true` (always assume changed); see [AnimChannelFixed.md](AnimChannelFixed.md) for the override that returns `false` |
| `virtual void output(std::ostream &out) const` | from `AnimGroup`; `ACMatrixSwitchType::output_value()` is used by matrix leaf classes to append the decomposed value |
| `get_name()` / `set_name()` | from `Namable`, via `AnimGroup` |

## Usage

```cpp
#include "animChannel.h"
#include "animChannelFixed.h"
#include "animBundle.h"
#include "luse.h"

// Build a tiny hierarchy: an AnimBundle root with one fixed matrix channel
// beneath it (AnimChannelFixed is the one concrete AnimChannelMatrix leaf
// covered in this doc set; table/dynamic subclasses are built similarly
// but load their per-frame data from an .egg file or a procedural source).
PT(AnimBundle) bundle = new AnimBundle("walk", 24.0f, 1);

LMatrix4 identity_xform = LMatrix4::ident_mat();
PT(AnimChannelFixed<ACMatrixSwitchType>) fixed_channel =
  new AnimChannelFixed<ACMatrixSwitchType>("root_joint", identity_xform);

// Treat it through the abstract AnimChannelMatrix interface:
AnimChannelMatrix *channel = fixed_channel.p();

LMatrix4 value;
channel->get_value(0, value);          // safe: pure-virtual, always implemented

LMatrix4 value_no_scale_shear;
channel->get_value_no_scale_shear(0, value_no_scale_shear);  // safe: falls back to get_value()

TypeHandle value_type = channel->get_value_type();  // TypeHandle for LMatrix4

// NOTE: channel->get_hpr(0, some_hpr) would hit nassertv(false) here,
// because AnimChannelFixed does not override the decomposition accessors.
```

## See also

[AnimChannelBase.md](AnimChannelBase.md), [AnimChannelScalar.md](AnimChannelScalar.md)
(the same template instantiated for scalar values — shorter page, cross-links
back here for the shared mechanics), [AnimChannelFixed.md](AnimChannelFixed.md)
(the constant-value leaf class used in the Usage example above)
