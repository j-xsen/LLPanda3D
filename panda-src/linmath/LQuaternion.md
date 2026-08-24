# LQuaternion

**Source:** `panda/src/linmath/lquaternion_src.h/.I/.cxx` (`f`/`d` only, no
`i` instantiation) · Library: `libp3linmath` · Notify category: `linmath`
**Inherits:** [`LVecBase4`](LVecBase.md)
**Inherited by:** [`LRotation`, `LOrientation`](LRotation.md)

`LQuaternion` represents a rotation (or, more precisely, an element of the
unit-quaternion group used to represent 3-D rotations without gimbal lock).
It's stored as an `LVecBase4` — `(r, i, j, k)` where `r` is the real/scalar
part and `(i, j, k)` the imaginary/vector part — so it inherits all of
`LVecBase4`'s componentwise arithmetic and gets `get_r()`/`get_i()`/
`get_j()`/`get_k()` as thin aliases over `get_x()`/`get_y()`/`get_z()`/
`get_w()` (index 0..3 in that order). See [LRotation.md](LRotation.md) for
the `LRotation`/`LOrientation` subclasses that give this the same name-only
semantic split that [LPoint](LPoint.md)/[LVector](LVector.md) give
`LVecBase`.

## Behavior notes

- **Quaternion multiplication composes rotations right-to-left, matching
  matrix convention when both are transformed against a row vector.**
  `operator*(const LQuaternion&)` computes the Hamilton product; there's no
  special normalization step — multiplying two non-unit quaternions produces
  a non-unit result, which most methods here otherwise assume won't happen
  (`LQuaternion` doesn't enforce unit length at the type level the way
  `LRotation`/`LOrientation`'s naming implies it should be used).
- **`normalize()` treats near-zero length as failure, same pattern as
  `LVecBase3::normalize()`** — sets everything to `0` and returns `false` on
  a (near-)zero quaternion, skips the divide if already unit length within
  `NEARLY_ZERO(FLOATTYPE)`.
- **`is_same_direction()`/`almost_same_direction()` are the correct way to
  compare two quaternions "as rotations," not `almost_equal()`.**
  `almost_equal()` is plain componentwise `(r,i,j,k)` comparison, which fails
  to recognize that `q` and `-q` represent the **same** rotation (unit
  quaternions double-cover `SO(3)`). `almost_same_direction(other, threshold)`
  instead checks whether `(*this) * invert(other)` is close to the identity
  quaternion — this correctly treats `q` and `-q` as equivalent.
  `is_same_direction()` is the zero-threshold (`NEARLY_ZERO`) convenience
  form. This is the "See Also" the [LVecBase3::get_standardized_hpr()](LVecBase.md)
  doc comment points to for a more reliable "are these two orientations the
  same" check than comparing Euler angles.
- **`set_hpr()`/`get_hpr()` are self-checking in debug builds via the
  `paranoid-hpr-quat` prc variable** (`config_linmath.h`'s
  `paranoid_hpr_quat`). When set and `NDEBUG` is not defined, `set_hpr()`
  independently computes the equivalent rotation via
  [`compose_matrix()`](ComposeMatrix.md) + `set_from_matrix()` and compares —
  if they disagree (beyond `almost_equal`, checked against both `compare`
  and `-compare` since either sign is a valid representation), it logs a
  `linmath_cat.warning()` and **overwrites `*this`** with the
  matrix-derived answer. `get_hpr()` does the same cross-check via
  `extract_to_matrix()` + [`decompose_matrix()`](ComposeMatrix.md). This
  variable exists because the direct quaternion↔hpr math and the
  matrix-mediated math both got independently implemented and needed a
  runtime way to confirm they still agree after later changes.
- **`set_hpr()` builds the quaternion as three axis-angle quaternions
  multiplied together** — `quat_h` about `LVector3::up(cs)`, `quat_p` about
  `LVector3::right(cs)`, `quat_r` about `LVector3::forward(cs)` (see
  [LVector.md](LVector.md) for how those three axes depend on `cs`) — each
  built directly from `sin(angle/2)`/`cos(angle/2)` rather than going through
  `set_from_axis_angle()`.
- **`get_axis()` vs. `get_axis_normalized()`**: `get_axis()` returns the raw
  imaginary part `(i, j, k)` (whose length encodes `sin(angle/2)`, not `1`);
  `get_axis_normalized()` divides it down to a true unit rotation axis. Use
  the normalized form unless you specifically want the unnormalized vector
  (e.g. for further quaternion algebra where renormalizing would be wasted
  work).
- **`get_up()`/`get_right()`/`get_forward()` are the quaternion's rotated
  basis vectors**, i.e. `xform(LVector3::up(cs))` etc. — the standard way to
  answer "which way is this orientation currently facing" without going
  through a matrix.
- **`is_identity()` vs. `is_almost_identity(tolerance)`**: `is_identity()` is
  exact-equality against `ident_quat()` (`(1,0,0,0)`); `is_almost_identity()`
  takes an explicit tolerance and is what `almost_same_direction()` is built
  on internally.
- **`extract_to_matrix()`/`set_from_matrix()` overload on both `LMatrix3`
  and `LMatrix4`** — going through `LMatrix4` just operates on its
  `get_upper_3()`/`set_upper_3()` block; the real conversion math lives in
  the `LMatrix3` overloads (out-of-line, `.cxx`).
- **`operator*(const LMatrix3&)`/`operator*(const LMatrix4&)` combine a
  quaternion with a matrix by converting the quaternion to a matrix first**
  and matrix-multiplying — not a fused/optimized quaternion-times-matrix
  operation.
- **`pure_imaginary(const LVector3&)`** builds a quaternion with `r=0` and
  the given vector as the imaginary part — used internally (and by callers
  doing quaternion-based vector rotation manually) as an intermediate value,
  not itself a valid rotation (zero real part means it's not unit-length in
  the way a rotation quaternion should be).

## API

### Construction
| Signature | Notes |
|---|---|
| `LQuaternion()` | Uninitialized |
| `LQuaternion(const LVecBase4 &copy)` | Reinterprets an `(r,i,j,k)` tuple |
| `LQuaternion(FLOATTYPE r, const LVecBase3 &copy)` / `LQuaternion(FLOATTYPE r, FLOATTYPE i, FLOATTYPE j, FLOATTYPE k)` | |
| `static LQuaternion pure_imaginary(const LVector3&)` | `r=0`; see behavior notes |
| `static const LQuaternion &ident_quat()` | Cached `(1,0,0,0)` |

### Component access
| Signature | Notes |
|---|---|
| `get_r/i/j/k()` / `set_r/i/j/k(FLOATTYPE)` | Aliases over `LVecBase4`'s `x/y/z/w` |
| `LVector3 get_axis() const` / `get_axis_normalized() const` | Raw vs. unit rotation axis |
| `FLOATTYPE get_angle_rad() const` / `get_angle() const` | Rotation angle (radians/degrees) |
| `LVector3 get_up/get_right/get_forward(CoordinateSystem cs = CS_default) const` | Rotated basis vectors |

### Construction from / conversion to other rotation representations
| Signature | Notes |
|---|---|
| `void set_from_axis_angle_rad(FLOATTYPE, const LVector3&)` / `set_from_axis_angle(FLOATTYPE deg, const LVector3&)` | |
| `void set_hpr(const LVecBase3 &hpr, CoordinateSystem cs = CS_default)` / `LVecBase3 get_hpr(CoordinateSystem cs = CS_default) const` | Self-checking under `paranoid-hpr-quat`; see behavior notes |
| `void set_from_matrix(const LMatrix3&)` / `set_from_matrix(const LMatrix4&)` | |
| `void extract_to_matrix(LMatrix3&) const` / `extract_to_matrix(LMatrix4&) const` | |

### Algebra
| Signature | Notes |
|---|---|
| `LQuaternion conjugate() const` / `bool conjugate_from(const LQuaternion&)` / `conjugate_in_place()` | Negates the imaginary part |
| `bool invert_from(const LQuaternion&)` / `invert_in_place()` | For a unit quaternion, equals the conjugate |
| `LQuaternion multiply(const LQuaternion&) const` / `operator*(const LQuaternion&) const` / `operator*=` | Hamilton product |
| `operator+` / `operator-` / `operator-()` (unary) | Componentwise, inherited-style |
| `operator*(FLOATTYPE)` / `operator/(FLOATTYPE)` | Scalar scale |
| `LQuaternion __pow__(FLOATTYPE) const` | Fractional/repeated rotation |
| `bool normalize()` | See behavior notes |
| `LVecBase3 xform(const LVecBase3&) const` / `LVecBase4 xform(const LVecBase4&) const` | Rotate a vector by this quaternion |

### Comparison
| Signature | Notes |
|---|---|
| `bool almost_equal(const LQuaternion&) const` / `almost_equal(const LQuaternion&, FLOATTYPE threshold) const` | Plain componentwise — does **not** treat `q`/`-q` as equal |
| `bool is_same_direction(const LQuaternion&) const` / `almost_same_direction(const LQuaternion&, FLOATTYPE threshold) const` | Rotation-equivalence check — treats `q`/`-q` as equal; prefer this over `almost_equal` for "same rotation?" |
| `FLOATTYPE angle_rad(const LQuaternion&) const` / `angle_deg(...)` | Angle between two rotations |
| `bool is_identity() const` / `is_almost_identity(FLOATTYPE tolerance) const` | |

### Free functions
| Signature | Notes |
|---|---|
| `LQuaternion invert(const LQuaternion&)` | Non-mutating |
| `LMatrix3 operator*(const LMatrix3&, const LQuaternion&)` / `LMatrix4 operator*(const LMatrix4&, const LQuaternion&)` | Matrix-times-quaternion, via matrix conversion |

## Usage

```cpp
LQuaternion q;
q.set_hpr(LVecBase3(90, 0, 0), CS_zup_right);   // 90-degree heading turn

LVector3 forward = q.xform(LVector3::forward(CS_zup_right));

LQuaternion a, b;
a.set_from_axis_angle(45, LVector3(0, 0, 1));
b.set_from_axis_angle(-45, LVector3(0, 0, 1));
bool same_rotation = a.is_same_direction(b.conjugate());  // true
```

## See also

[LRotation.md](LRotation.md) (`LRotation`/`LOrientation` — semantic
subclasses, no new storage) · [LVecBase.md](LVecBase.md) (storage base;
`get_standardized_hpr()` doc points here) · [LMatrix.md](LMatrix.md)
(`extract_to_matrix`/`set_from_matrix` counterpart) ·
[ComposeMatrix.md](ComposeMatrix.md) (what `paranoid-hpr-quat` cross-checks
against) · [CoordinateSystem.md](CoordinateSystem.md) · [README.md](README.md)
