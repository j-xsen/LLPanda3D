# LVector2 / LVector3 / LVector4

**Source:** `panda/src/linmath/lvector2_src.h/.I/.cxx`, `lvector3_src.*`,
`lvector4_src.*` (each `#include`d three times — `f`/`d`/`i` — by
`lvector2.h`/`lvector3.h`/`lvector4.h`) · plus `lvector2_ext_src.*` /
`lvector3_ext_src.*` / `lvector4_ext_src.*` (Python `__repr__`/swizzle
plumbing) · Library: `libp3linmath` · Notify category: `linmath`
**Inherits:** [`LVecBase2`/`LVecBase3`/`LVecBase4`](LVecBase.md)
**Inherited by:** (none)

`LVector2`/`LVector3`/`LVector4` represent a **direction and distance** — a
displacement between two points, or a free-floating direction like a surface
normal — as opposed to [LPoint](LPoint.md)'s specific location. Same storage
as [LVecBase](LVecBase.md), same size; the type exists to get compile-time
point/vector arithmetic rules (see [LPoint.md](LPoint.md)'s behavior notes
for the full point/vector return-type table) and to host vector-only
operations like angle-between and the coordinate-system-relative named
constructors below.

## Behavior notes

- **`zero()`/`unit_x()`/`unit_y()`/`unit_z()` are reinterpret-casts of the
  base class's static consts, not separate storage.** `LVector3::zero()`
  returns `(const LVector3 &)LVecBase3::zero()` — safe only because
  `LVector3` adds no data members over `LVecBase3` (same is true for
  [LPoint](LPoint.md)'s equivalents); don't take this as license to add
  fields to a subclass elsewhere in this hierarchy.
- **`LVector3` has three coordinate-system-aware named constructors that
  `LVector2`/`LVector4` don't:** `up(cs)`, `right(cs)`, `forward(cs)` (plus
  `down(cs)`/`left(cs)`/`back(cs)`, their negations) and `rfu(right, fwd, up,
  cs)`. These are how code writes "a unit vector pointing up" without hard-
  coding whether "up" is `+Z` or `+Y` for the active
  [CoordinateSystem](CoordinateSystem.md):
  - `right(cs)` is always `(1, 0, 0)` regardless of `cs` — X is always
    "right" in every Panda coordinate system.
  - `up(cs)` is `(0, 0, 1)` for `CS_zup_right`/`CS_zup_left`, `(0, 1, 0)` for
    `CS_yup_right`/`CS_yup_left`.
  - `forward(cs)` depends on **both** up-axis and handedness: `(0, 1, 0)` for
    `CS_zup_right`, `(0, -1, 0)` for `CS_zup_left`, `(0, 0, -1)` for
    `CS_yup_right`, `(0, 0, 1)` for `CS_yup_left`.
  - `rfu(right_v, fwd_v, up_v, cs)` combines all three into one vector — the
    fast path in `lvector3_src.I` special-cases each of the four coordinate
    systems directly rather than actually computing
    `right(cs)*right_v + forward(cs)*fwd_v + up(cs)*up_v` (the comment in the
    source spells out that this is what it's logically equivalent to).
  - `LPoint3::origin(cs)`/`LPoint3::rfu(...)` (see [LPoint.md](LPoint.md))
    are built directly on these.
- **`angle_rad()` deliberately avoids `acos(dot(a, b))`** — the comment in
  `lvector3_src.I` notes `acos` "behaves poorly as `dot()` approaches 1.0"
  (near-parallel vectors), so it instead computes `2 * asin(|a±b| / 2)`,
  choosing `+` or `-` based on the sign of the dot product to stay in the
  numerically stable half of `asin`'s domain.
- **`signed_angle_rad(other, ref)` needs a third "reference" vector to
  disambiguate sign** — the unsigned `angle_rad()` result is negated if
  `cross(other).dot(ref) < 0`, i.e. if the rotation from `this` to `other`
  is clockwise when viewed from the `ref` direction. `LVector2`'s
  `signed_angle_rad(other)` (no `ref` needed — 2D rotation direction is
  unambiguous) is a distinct, simpler two-argument overload.
- **`relative_angle_rad()`/`relative_angle_deg()` are explicitly
  `@deprecated`** per their doc comments in `lvector3_src.I` — present for
  compatibility but callers should use `angle_rad()`/`signed_angle_rad()`
  instead.
- **`normalized()`/`project()`/the angle family are all gated
  `#ifndef FLOATTYPE_IS_INT`** — `LVector3i` has none of them, same pattern
  as [LVecBase](LVecBase.md).
- **Point/vector return-type rules mirror [LPoint](LPoint.md)'s, inverted:**
  `LVector3::operator+(const LVecBase3&)` returns `LVecBase3` (demotes),
  `operator+(const LVector3&)` returns `LVector3` (vector + vector = vector).
  There's no `vector + point` overload on `LVector3` itself — that's
  `LPoint3::operator+(const LVector3&)`, defined on the point side.

## API

Only what's new or different from [LVecBase](LVecBase.md) is shown; every
inherited method is available unchanged. Shown for `LVector3`;
`LVector2`/`LVector4` are the same shape minus the `LVector3`-only
angle/named-constructor methods.

### Construction
| Signature | Notes |
|---|---|
| `LVector3()` | Uninitialized |
| `LVector3(const LVecBase3 &copy)` | Promotes a base vector |
| `LVector3(FLOATTYPE fill_value)` / `LVector3(FLOATTYPE x, FLOATTYPE y, FLOATTYPE z)` | |
| `LVector3(const LVecBase2 &copy, FLOATTYPE z)` | Widen by one component |
| `static const LVector3 &zero() / unit_x() / unit_y() / unit_z()` | Reinterpret-casts of the `LVecBase3` statics |

### Vector-typed arithmetic
| Signature | Notes |
|---|---|
| `LVecBase3 operator+(const LVecBase3&) const` / `LVector3 operator+(const LVector3&) const` | Demotes vs. stays a vector |
| `LVecBase3 operator-(const LVecBase3&) const` / `LVector3 operator-(const LVector3&) const` | Same pattern |
| `LVector3 operator-() const` / `operator*(FLOATTYPE)` / `operator/(FLOATTYPE)` | |
| `LVector3 cross(const LVecBase3&) const` | Returns `LVector3` (contrast `LPoint3::cross`, which returns `LPoint3`) |
| `LVector3 normalized() const` / `LVector3 project(const LVecBase3&) const` | Non-`i` only |

### Angle (non-`i` only, `LVector3`)
| Signature | Notes |
|---|---|
| `FLOATTYPE angle_rad(const LVector3&) const` / `angle_deg(...)` | Unsigned angle between two vectors; numerically stable near-parallel |
| `FLOATTYPE signed_angle_rad(const LVector3 &other, const LVector3 &ref) const` / `signed_angle_deg(...)` | Signed, using `ref` to determine rotation direction |
| `FLOATTYPE relative_angle_rad(const LVector3&) const` / `relative_angle_deg(...)` | **Deprecated** — use `angle_rad`/`signed_angle_rad` |

### Angle (non-`i` only, `LVector2`)
| Signature | Notes |
|---|---|
| `FLOATTYPE signed_angle_rad(const LVector2&) const` / `signed_angle_deg(...)` | No `ref` parameter needed in 2D |

### Coordinate-system-relative named constructors (`LVector3` only)
| Signature | Notes |
|---|---|
| `static LVector3 up/right/forward(CoordinateSystem cs = CS_default)` | See behavior notes for exact axis mapping per `cs` |
| `static LVector3 down/left/back(CoordinateSystem cs = CS_default)` | Negations of the above |
| `static LVector3 rfu(FLOATTYPE right, FLOATTYPE fwd, FLOATTYPE up, CoordinateSystem cs = CS_default)` | Combined displacement in right/forward/up terms |

## Usage

```cpp
// build a camera-relative offset without hard-coding the up axis
LVector3 fwd = LVector3::forward(CS_zup_right);
LVector3 offset = LVector3::rfu(0, -5, 2, CS_zup_right);  // 5 back, 2 up

LVector3 a(1, 0, 0), b(0, 1, 0);
float angle = a.angle_deg(b);              // 90
float signed_a = a.signed_angle_deg(b, LVector3::up(CS_zup_right));
```

## See also

[LVecBase.md](LVecBase.md) (shared storage/arithmetic base) ·
[LPoint.md](LPoint.md) (the location counterpart; full point/vector
return-type table) · [CoordinateSystem.md](CoordinateSystem.md) (`CS_zup_right`
etc., and `is_right_handed()`) · [LMatrix.md](LMatrix.md) (`xform_vec()`
ignores translation, unlike `xform_point()`) · [README.md](README.md)
