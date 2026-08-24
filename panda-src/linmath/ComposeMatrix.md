# compose_matrix / decompose_matrix

**Source:** `panda/src/linmath/compose_matrix_src.h/.I/.cxx` (`f`/`d`,
`#include`d twice by `compose_matrix.h`) + `compose_matrix.h/.cxx` (the
`num_matrix_components` constant and `matrix_component_letters`/
`matrix_component_defaults` arrays live in the outer, non-macro-instantiated
file) · Library: `libp3linmath` · Notify category: `linmath`

Free functions (not a class) that convert between an `LMatrix3`/`LMatrix4`
and its separate **scale / shear / hpr (heading-pitch-roll) / translate**
components. This is how Panda's scene graph (see
[TransformState.md](../pgraph/TransformState.md)) presents a node's
transform as independently settable pos/hpr/scale properties while storing
(or caching) the composed matrix underneath, and it's the standard way
agent-generated code should build a transform matrix rather than hand-writing
rotation/scale/translation matrix multiplication.

## Behavior notes

- **`compose_matrix()`/`decompose_matrix()` come in three input/output
  shapes, all overloads of the same name:** individual `LVecBase3`
  scale/shear/hpr/translate out-parameters, a flat `FLOATTYPE
  components[num_matrix_components]` array (`num_matrix_components == 12`,
  defined `static const int` in `compose_matrix.h`), and (for `LMatrix3`, no
  translate) the same without the translate component. All three ultimately
  call through to the same `LMatrix3`-based core (`decompose_matrix(mat,
  scale, shear, hpr, cs)` operating on the 3x3 case; the `LMatrix4` overloads
  just also pull/push `get_row3(3)`/build via `LMatrix4(upper3, translate)`).
- **The 12-float array layout is `[scale.x,y,z, shear.x,y,z, hpr.h,p,r,
  translate.x,y,z]`**, in that fixed order — matching
  `matrix_component_letters` (`compose_matrix.cxx`): the string
  `"ijkabchprxyz"`, one letter per array slot (`i,j,k` = scale, `a,b,c` =
  shear, `h,p,r` = hpr, `x,y,z` = translate) — used by config-file parsing
  code elsewhere in the engine (e.g. `AnimChannelMatrixXfmTable`) that needs
  to know which array index is which named component.
  `matrix_component_defaults` is `{1,1,1, 0,0,0, 0,0,0, 0,0,0}` — identity
  scale, zero everything else.
- **hpr is heading/pitch/roll in degrees, applied as three axis rotations
  composed via [LQuaternion](LQuaternion.md)-equivalent half-angle math** —
  heading about `LVector3::up(cs)`, pitch about `LVector3::right(cs)`, roll
  about `LVector3::forward(cs)`, exactly mirroring
  `LQuaternion::set_hpr()`'s construction (see [LQuaternion.md](LQuaternion.md)) —
  the two are meant to and (per the `paranoid-hpr-quat` cross-check
  documented there) are verified to agree.
- **Deprecated overloads without a `shear` parameter still exist** for
  source compatibility — `compose_matrix(mat, scale, hpr, cs)` (no shear)
  just calls the shear-taking version with `shear = zero()`;
  `decompose_matrix(mat, scale, hpr, cs)` (no shear) calls the full version
  and then **fails** (`false`) if the decomposed shear isn't
  `almost_equal(zero())` — i.e. the no-shear decompose silently rejects any
  matrix that actually has shear in it, rather than losing that information
  silently.
- **`decompose_matrix()` can fail and returns `bool`** — not every 3x3/4x4
  matrix decomposes losslessly into scale/shear/hpr/translate (e.g. a
  matrix with negative determinant / mirroring, or a genuinely projective
  4x4 matrix, doesn't fit this affine decomposition model). Always check the
  return value; on failure the out-parameters are left partially written.
- **`decompose_matrix_old_hpr()`/`old_to_new_hpr()` are explicitly marked
  "transitional... migrate code from the old, incorrect hpr calculations"**
  in the header comment — present only so old data/save files using the
  previous (buggy) hpr convention can be converted forward. New code should
  never call `decompose_matrix_old_hpr()` directly; use the unqualified
  `decompose_matrix()` and, if migrating legacy data, `old_to_new_hpr()` on
  the result.
- **A static helper, `unwind_yup_rotation_old_hpr()` (file-local to
  `compose_matrix_src.cxx`, not exposed in the header)**, is part of that old
  decomposition path — it assumes a **right-handed Y-up** coordinate system
  regardless of the matrix's actual coordinate system, projecting each axis
  onto a plane and using `atan2` to peel off roll, then heading, then pitch
  in turn, "unwinding" the rotation from the matrix as it goes. It's
  mentioned here only because an agent reading `compose_matrix_src.cxx`
  directly might otherwise mistake it for the current algorithm.

## API

| Signature | Notes |
|---|---|
| `void compose_matrix(LMatrix3 &mat, const LVecBase3 &scale, const LVecBase3 &shear, const LVecBase3 &hpr, CoordinateSystem cs = CS_default)` | 3x3, no translate |
| `void compose_matrix(LMatrix4 &mat, const LVecBase3 &scale, const LVecBase3 &shear, const LVecBase3 &hpr, const LVecBase3 &translate, CoordinateSystem cs = CS_default)` | Full affine 4x4 |
| `void compose_matrix(LMatrix4 &mat, const FLOATTYPE components[12], CoordinateSystem cs = CS_default)` | Flat-array form; layout = scale, shear, hpr, translate |
| `bool decompose_matrix(const LMatrix3 &mat, LVecBase3 &scale, LVecBase3 &shear, LVecBase3 &hpr, CoordinateSystem cs = CS_default)` | Returns `false` if `mat` isn't decomposable this way |
| `bool decompose_matrix(const LMatrix4 &mat, LVecBase3 &scale, LVecBase3 &shear, LVecBase3 &hpr, LVecBase3 &translate, CoordinateSystem cs = CS_default)` | |
| `bool decompose_matrix(const LMatrix4 &mat, FLOATTYPE components[12], CoordinateSystem cs = CS_default)` | Flat-array form |
| `void compose_matrix(LMatrix3&, scale, hpr, cs)` / `compose_matrix(LMatrix4&, scale, hpr, translate, cs)` | **Deprecated**, no-shear overloads (shear = 0) |
| `bool decompose_matrix(LMatrix3&, scale, hpr, cs)` / `decompose_matrix(LMatrix4&, scale, hpr, translate, cs)` | **Deprecated**, fails if the matrix actually has shear |
| `bool decompose_matrix_old_hpr(const LMatrix3&, scale, shear, hpr, cs)` | Legacy hpr convention — for migrating old data only, see behavior notes |
| `LVecBase3 old_to_new_hpr(const LVecBase3 &old_hpr)` | Converts a legacy hpr triple to the current convention |

### Constants (`compose_matrix.h`, outside the macro-instantiated part)
| Name | Value | Notes |
|---|---|---|
| `num_matrix_components` | `12` | Size of the flat-array compose/decompose overloads |
| `matrix_component_letters` | `"ijkabchprxyz"` | One letter per array slot: scale, shear, hpr, translate |
| `matrix_component_defaults` | `{1,1,1, 0,0,0, 0,0,0, 0,0,0}` | Identity-transform defaults for the array form |

## Usage

```cpp
LMatrix4 mat;
compose_matrix(mat,
               LVecBase3(2, 2, 2),      // scale
               LVecBase3(0, 0, 0),      // shear
               LVecBase3(90, 0, 0),     // hpr
               LVecBase3(0, 5, 0),      // translate
               CS_zup_right);

LVecBase3 scale, shear, hpr, translate;
if (decompose_matrix(mat, scale, shear, hpr, translate, CS_zup_right)) {
  // round-trips back to the original components (within floating-point error)
}
```

## See also

[LMatrix.md](LMatrix.md) (the matrices being built/decomposed) ·
[LQuaternion.md](LQuaternion.md) (`set_hpr`/`get_hpr` build on the same hpr
convention, cross-checked against this via `paranoid-hpr-quat`) ·
[CoordinateSystem.md](CoordinateSystem.md) (`cs` parameter throughout) ·
[TransformState.md](../pgraph/TransformState.md) (`make_pos_hpr_scale_shear()`
takes the same scale/shear/hpr/translate decomposition this module works
with) · [README.md](README.md)
