# LPoint2 / LPoint3 / LPoint4

**Source:** `panda/src/linmath/lpoint2_src.h/.I/.cxx`, `lpoint3_src.*`,
`lpoint4_src.*` (each `#include`d three times — `f`/`d`/`i` — by
`lpoint2.h`/`lpoint3.h`/`lpoint4.h`) · plus `lpoint2_ext_src.*` /
`lpoint3_ext_src.*` / `lpoint4_ext_src.*` (Python `__repr__`/swizzle
plumbing) · Library: `libp3linmath` · Notify category: `linmath`
**Inherits:** [`LVecBase2`/`LVecBase3`/`LVecBase4`](LVecBase.md)
**Inherited by:** (none)

`LPoint2`/`LPoint3`/`LPoint4` represent a **specific location** in space, as
opposed to [LVector](LVector.md)'s displacement/direction. They add no new
storage over [LVecBase](LVecBase.md) — same `_v` layout, same size — only a
different *type* so the compiler enforces point/vector arithmetic rules:
`point - point = vector`, `point + vector = point`, `vector + vector =
vector`, but `point + point` and `vector - point` are deliberately **not**
overloaded the way you might expect (see below). The type distinction alone
is Panda's whole reason for `LPoint`/`LVector`/`LVecBase` being three classes
instead of one — see the [module README](README.md)'s "Core concepts" for
why this matters when transforming by a matrix.

## Behavior notes

- **Mixed point/vector arithmetic returns whichever type is still
  semantically meaningful, and demotes to `LVecBase` when it isn't.**
  `LPoint3::operator+(const LVecBase3&)` returns `LVecBase3` (adding two
  "positions" isn't meaningful as a position), but
  `LPoint3::operator+(const LVector3&)` returns `LPoint3` (point + vector =
  point, still a position). Symmetrically, `operator-(const LVecBase3&)`
  returns `LVecBase3`, `operator-(const LPoint3&)` returns `LVector3` (the
  displacement between two points), and `operator-(const LVector3&)` returns
  `LPoint3` (point minus displacement = point). There is no
  `LPoint3::operator+(const LPoint3&)` overload distinct from the inherited
  `LVecBase3` one — adding two points falls through to the base class and
  yields a plain `LVecBase3`, which is a compile-time nudge that "sum of two
  positions" isn't a position.
- **Only `LPoint3` gets the named "origin"/`rfu` constructors** — `LPoint2`
  and `LPoint4` don't have coordinate-system-aware factory functions.
  `LPoint3::origin(CoordinateSystem cs = CS_default)` is just
  `LPoint3::zero()` regardless of `cs` (the origin is the origin in every
  coordinate system Panda supports); `LPoint3::rfu(right, fwd, up, cs)`
  builds a point offset from the origin by right/forward/up displacements —
  see [LVector.md](LVector.md)'s `rfu()` for how axis mapping depends on `cs`
  (Z-up vs. Y-up, left- vs. right-handed).
- **`normalized()`/`project()` are still defined on `LPoint3`/`LPoint2`/`LPoint4`**
  (non-`i` instantiations) even though "normalizing a point" is a slightly odd
  operation mathematically — it treats the point as a vector from the origin.
  There's no `length()`/`normalize()` override here beyond what
  [LVecBase](LVecBase.md) already provides; Panda doesn't restrict these to
  `LVector` only.
- **`get_xy()`/`get_xz()`/`get_yz()` (3→2) and `get_xyz()`/`get_xy()` (4→3/2)
  return `LPoint2`/`LPoint3`, not `LVecBase2`/`LVecBase3`** — slicing a point
  keeps it a point, unlike some of the arithmetic operators above that
  demote to the base class.
- **Construction accepts an `LVecBase` of the same arity implicitly**
  (`LPoint3(const LVecBase3 &copy)`), so an `LVecBase3` returned from an
  operator that demoted the type can be handed straight back into a
  constructor to "promote" it to a point again — this is the normal pattern
  for chaining arithmetic that would otherwise return the base type.
- **`LPoint3::cross(const LVecBase3&) const` returns `LPoint3`**, not
  `LVector3` — a quirk of the cross-product method not following the
  point/vector return-type convention used everywhere else in this class
  (cross product of two vectors is conventionally a vector, but the
  in-class override here is typed as `LPoint3`).
- **No integer-only special-casing beyond what `LVecBase` already does** —
  `LPoint2i`/`LPoint3i`/`LPoint4i` exist (see `intnames.h` in the [module
  README](README.md)) and simply lack the `#ifndef FLOATTYPE_IS_INT`-gated
  methods (`normalized()`, `project()`) that `LVecBase`'s int instantiation
  also lacks.

## API

Only what's new or different from [LVecBase](LVecBase.md) is shown; every
inherited method (`get_x`/`set_x`, `dot`, `length`, `compare_to`,
`almost_equal`, `output`, `write_datagram`/`read_datagram`, ...) is available
unchanged. Shown for `LPoint3`; `LPoint2`/`LPoint4` are the same shape minus
the `LPoint3`-only named constructors.

### Construction
| Signature | Notes |
|---|---|
| `LPoint3()` | Uninitialized |
| `LPoint3(const LVecBase3 &copy)` | Promotes a base vector to a point |
| `LPoint3(FLOATTYPE fill_value)` / `LPoint3(FLOATTYPE x, FLOATTYPE y, FLOATTYPE z)` | |
| `LPoint3(const LVecBase2 &copy, FLOATTYPE z)` | Widen by one component |
| `static const LPoint3 &zero() / unit_x() / unit_y() / unit_z()` | |

### Point/vector-aware arithmetic
| Signature | Notes |
|---|---|
| `LVecBase3 operator+(const LVecBase3&) const` | Demotes — "point + arbitrary vecbase" isn't a point |
| `LPoint3 operator+(const LVector3&) const` | Point + displacement = point |
| `LVecBase3 operator-(const LVecBase3&) const` | Demotes |
| `LVector3 operator-(const LPoint3&) const` | Point − point = displacement |
| `LPoint3 operator-(const LVector3&) const` | Point − displacement = point |
| `LPoint3 operator-() const` / `operator*(FLOATTYPE)` / `operator/(FLOATTYPE)` | Stay `LPoint3` |
| `LPoint3 cross(const LVecBase3&) const` | Returns `LPoint3`, not `LVector3` — see behavior notes |
| `LPoint3 normalized() const` / `LPoint3 project(const LVecBase3&) const` | Non-`i` only |

### Slicing
| Signature | Notes |
|---|---|
| `LPoint2 get_xy/xz/yz() const` (`LPoint3`) · `LPoint3 get_xyz() const`, `LPoint2 get_xy() const` (`LPoint4`) | Returns `LPoint`, not `LVecBase` |

### `LPoint3`-only named constructors
| Signature | Notes |
|---|---|
| `static const LPoint3 &origin(CoordinateSystem cs = CS_default)` | `= zero()`, `cs` accepted but doesn't change the result |
| `static LPoint3 rfu(FLOATTYPE right, FLOATTYPE fwd, FLOATTYPE up, CoordinateSystem cs = CS_default)` | Displacement from the origin in right/forward/up terms; see [LVector.md](LVector.md) for the axis mapping |

## Usage

```cpp
LPoint3 a(0, 0, 0);
LPoint3 b(3, 4, 0);

LVector3 displacement = b - a;        // LVector3, not LPoint3
float dist = displacement.length();

LPoint3 midpoint(a + displacement * 0.5f);  // LVecBase3 promoted back to LPoint3

LPoint3 spawn = LPoint3::rfu(0, 5, 0, CS_zup_right);  // 5 units "forward" of the origin
```

## See also

[LVecBase.md](LVecBase.md) (shared storage/arithmetic base) ·
[LVector.md](LVector.md) (the displacement counterpart; `rfu`/`up`/`right`/`forward`
axis-mapping details live there) · [LMatrix.md](LMatrix.md) (`xform_point()`
applies the matrix's translation; `xform_vec()` does not) ·
[TransformState.md](../pgraph/TransformState.md) (composes an `LPoint3` pos +
`LVecBase3` scale + rotation into a node's transform) · [README.md](README.md)
