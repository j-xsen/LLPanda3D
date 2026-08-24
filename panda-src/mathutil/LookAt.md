# heads_up() / look_at() / rotate_to()

**Source:** `panda/src/mathutil/look_at_src.h/.I/.cxx` (macro-templated body) +
`look_at.h` (float/double wrapper) · `rotate_to_src.cxx` + `rotate_to.h`
(explicit float/double overloads, no macro instantiation)
**Inherits:** n/a — free functions
**Inherited by:** n/a

Free functions that build a rotation (as an `LMatrix3`, `LMatrix4`, or
`LQuaternion`) from a facing direction, the standard building block for
"orient this object to face that point/direction" logic (billboards,
character facing, projectile orientation). `rotate_to()` solves a related
but distinct problem: the rotation that takes one specific vector onto
another, by the shortest angular path.

`look_at_src.h/.I/.cxx` follow the same `fltnames.h`/`dblnames.h`
macro-instantiation pattern as [Plane.md](Plane.md)/[Frustum.md](Frustum.md)
(see [../linmath/README.md](../linmath/README.md)). `rotate_to_src.cxx` is
included by `rotate_to.h` the same two ways but is a much smaller
translation unit with no separate `.I`.

## Behavior notes

- **`heads_up()` and `look_at()` differ only in which input vector "wins"
  when `fwd` and `up` aren't exactly perpendicular.** `look_at()` rotates
  `fwd` to exactly the standard forward axis first, then fits `up` as close
  to the standard up axis as the (now-fixed) forward rotation allows.
  `heads_up()` does the reverse: `up` is matched exactly, `fwd` is fit as
  closely as possible around it. If the two vectors *are* perpendicular,
  both functions produce the identical result — the distinction only
  matters for imprecise/lazy call sites that pass a not-quite-perpendicular
  `up` (e.g. always passing world-up even when looking nearly straight
  up/down).
  Practical rule of thumb baked into the two names: use `look_at()` when
  the "at" target matters (a turret aiming at an enemy) and `heads_up()`
  when staying upright matters more (a character or camera that must not
  tilt, even if that means "at" drifts slightly).
- **Every overload ultimately funnels through the `LMatrix3` implementation
  in `look_at_src.cxx`** — the `LMatrix4` overloads build an `LMatrix3`
  then wrap it (zero translation row/column implied); the `LQuaternion`
  overloads build an `LMatrix3` then call `LQuaternion::set_from_matrix()`.
  There is no independent quaternion-native derivation.
- **Both functions branch on `CoordinateSystem` between "Z-up" and "Y-up"
  families early, then handle right/left-handed within each family by
  flipping signs on specific intermediate terms** (see the `cs ==
  CS_zup_right`/`else` branches inside each `.cxx`) rather than by a
  generic post-multiply with a handedness-conversion matrix — a
  hand-optimized derivation, matching [Frustum.md](Frustum.md)'s
  projection-matrix approach.
- **Degenerate inputs (zero-length projected vectors, from an `up` parallel
  to `fwd`, etc.) fall back to an arbitrary default 2-D basis
  (`FLOATNAME(LVector2)(0.0f, 1.0f)`) rather than asserting or producing
  NaN** — the resulting rotation in that exact edge case is not
  necessarily meaningful, but the function always returns *some* valid,
  finite rotation matrix.
- **`rotate_to(mat, a, b)` assumes both `a` and `b` are already
  normalized** — it does not normalize them itself (the header comment
  states this explicitly); passing non-unit vectors produces a wrong
  (non-rotation) result silently.
- **`rotate_to()`'s collinear-vector handling is asymmetric**: if `a` and
  `b` point the same direction (`cos_theta >= 0`, and the cross product's
  length is below a `0.0001` threshold), it returns the identity matrix
  directly. If they point exactly opposite (`cos_theta < 0`), a 180°
  rotation needs *some* perpendicular axis to rotate around (infinitely
  many exist), so it picks one deterministically: the world axis
  (X/Y/Z-unit vector) most nearly perpendicular to `a`, found by comparing
  `|a.x|`, `|a.y|`, `|a.z|` and choosing whichever is *smallest* (least
  aligned with `a`, hence most reliably perpendicular after the
  cross-product step).
- **The core rotation-matrix formula is Rodrigues' rotation formula**
  (`t = 1 - cos_theta`, then the standard `axis*axis^T*t + cos_theta*I +
  sin_theta*[axis]_x` construction) — chooses the axis as the (normalized)
  cross product of `a` and `b`, which by construction gives the *smallest*
  possible rotation angle taking `a` to `b`.

## API

### heads_up() / look_at() (all `BEGIN_PUBLISH`/`END_PUBLISH`)
| Signature | Notes |
|---|---|
| `void heads_up(LMatrix3 &mat, const LVector3 &fwd, const LVector3 &up = LVector3::up(), CoordinateSystem cs = CS_default)` | Matches `up` exactly |
| `void look_at(LMatrix3 &mat, const LVector3 &fwd, const LVector3 &up = LVector3::up(), CoordinateSystem cs = CS_default)` | Matches `fwd` exactly |
| `void heads_up(LMatrix4 &mat, ...)` / `void look_at(LMatrix4 &mat, ...)` | Same signatures, `LMatrix4` output (zero translation) |
| `void heads_up(LQuaternion &quat, ...)` / `void look_at(LQuaternion &quat, ...)` | Same signatures, quaternion output |

### rotate_to()
| Signature | Notes |
|---|---|
| `void rotate_to(LMatrix3f &mat, const LVector3f &a, const LVector3f &b)` / `LMatrix3d` overload | `a`/`b` must already be unit-length |
| `void rotate_to(LMatrix4f &mat, const LVector3f &a, const LVector3f &b)` / `LMatrix4d` overload | Wraps the `LMatrix3` result |

## Usage

```cpp
LVector3 to_target = target_pos - my_pos;
LQuaternion facing;
look_at(facing, to_target);   // exactly faces target, up may drift a bit

LMatrix3 spin;
rotate_to(spin, LVector3::forward(), to_target);  // shortest rotation onto to_target
```

## See also

[Frustum.md](Frustum.md) (same coordinate-system-branching style) ·
[../linmath/README.md](../linmath/README.md) · [README.md](README.md)
