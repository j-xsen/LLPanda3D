# LMatrix3 / LMatrix4

**Source:** `panda/src/linmath/lmatrix3_src.h/.I/.cxx`, `lmatrix4_src.*`
(`lmatrix3_src.h` `#include`s `lmatrix4_src.h` internally so both are
generated together from `lmatrix.h`'s two `#include "lmatrix3_src.h"`
lines — `f`/`d` only, no `i` instantiation) · plus `lmatrix3_ext_src.*` /
`lmatrix4_ext_src.*` (Python `__reduce__`/`__repr__`) and `lmat_ops_src.*`
(free-function `vec * matrix` operators) · Library: `libp3linmath` ·
Notify category: `linmath`
**Inherits:** none (value types; register a `TypeHandle`)
**Inherited by:** (none)

`LMatrix3` is a 3x3 row-major matrix, typically either a full 2-D affine
transform (rotate/scale/translate, e.g. for a texture matrix) or the
rotate/scale-only upper-left block of a 3-D transform. `LMatrix4` is the
full 4x4 row-major matrix used for 3-D affine and projective transforms.
**Vectors are row vectors multiplied on the left of the matrix**
(`v * M`, not `M * v`) — see `lmat_ops_src.h`'s operators and the free
`operator*(const LVecBase4&, const LMatrix4&)` — which is why a translation
lives in the matrix's **fourth row**, not its fourth column, and why
`operator*=`-style chaining reads left-to-right in transform order
(`child_to_world = child_to_parent * parent_to_world`, not the reverse).

## Behavior notes

- **`LMatrix4` is 16-byte aligned for SSE2; `LMatrix3` is not.** Same
  reasoning as [LVecBase4](LVecBase.md) — `ALIGN_LINMATH` + `LINMATH_MATRIX`
  for `_m` on `LMatrix4`, `UNALIGNED_LINMATH_MATRIX` on `LMatrix3` (too few
  floats to be worth it). `UnalignedLMatrix4` (declared inside
  `lmatrix4_src.h`, no separate file) is the tightly-packed counterpart, same
  relationship as `UnalignedLVecBase4` to `LVecBase4` — construct one when
  you need to pack 16 floats with no padding, then convert to `LMatrix4` to
  actually use it.
- **Vector/point transforms come in "fast" and "general" pairs.**
  `xform_vec()`/`xform_point()` assume the matrix is affine (`LMatrix4`) or
  orthonormal (`LMatrix3::xform_vec`) and skip the perspective divide /
  inverse-transpose that would otherwise be needed; `xform_vec_general()`/
  `xform_point_general()` are the fully-general forms — `xform_point_general()`
  appends `w=1`, transforms as a 4-vector, and divides by the resulting `w`
  (handles perspective matrices correctly), while `xform_vec_general()`
  transforms by the **inverse transpose** of the upper 3x3 (correct for
  non-uniform scale / shear, where the plain rotation-only transform would
  give a wrong direction for e.g. a surface normal). Use the general forms
  whenever the matrix might not be a pure rotate+uniform-scale+translate.
- **`xform_point()` applies translation; `xform_vec()` does not** — this is
  the concrete mechanism behind [LPoint](LPoint.md)/[LVector](LVector.md)'s
  semantic split, and the reason `operator*(const LPoint3&, const LMatrix4&)`
  and `operator*(const LVector3&, const LMatrix4&)` in `lmat_ops_src.h` are
  separate overloads rather than one function on the common base.
- **`invert_from()` special-cases the affine case for speed.** Without Eigen,
  it first checks whether the matrix's last column is `(0, 0, 0, 1)` (i.e.
  no projective component) and if so delegates to `invert_affine_from()`,
  which only needs to invert the upper-left 3x3 (via `LMatrix3::invert_from`)
  and negate the translation — much cheaper than full 4x4 Gauss-Jordan via
  `decompose_mat()`/`back_sub_mat()`. With Eigen, `computeInverseWithCheck()`
  is used directly with a determinant threshold of `NEARLY_ZERO(FLOATTYPE)²`.
- **Inverting a singular matrix does not throw or leave garbage — it resets
  to identity and returns `false`.** Both paths, on failure, set `*this =
  ident_mat()`, log a `linmath_cat.warning()` (debug builds only), and
  `nassertr(!no_singular_invert, false)` — meaning if the
  `no-singular-invert` prc variable (see the [module README](README.md)) is
  set, this becomes a hard assertion failure instead of a silent
  identity-and-`false`. Always check the `bool` return value of
  `invert_from()`/`invert_in_place()` if the matrix might be singular.
- **`set_rotate_mat_normaxis()`/`rotate_mat_normaxis()` skip normalizing the
  axis argument** — the plain `set_rotate_mat()`/`rotate_mat()` normalize the
  given axis first; the `_normaxis` variants assume the caller has already
  passed a unit vector and skip that step, trading a safety check for speed.
  Passing a non-unit axis to the `_normaxis` form silently produces a
  skewed/scaled rotation, not an error.
- **Rotation direction flips with handedness, not just up-axis.** Both
  `set_rotate_mat(angle, axis, cs)` and `set_rotate_mat_normaxis()` negate
  `angle` when `IS_LEFT_HANDED_COORDSYSTEM(cs)` — "counterclockwise" swaps
  meaning between right- and left-handed [coordinate systems](CoordinateSystem.md),
  so the same stored angle produces a visually different rotation depending
  on `cs`.
- **`accumulate(other, weight)` computes `*this += other * weight`** — an
  animation-blending primitive (weighted sum of matrices), not a matrix
  product; the caller is responsible for whatever normalization/renormalization
  the blended result needs afterward.
- **`get_upper_3()`/`set_upper_3()` isolate the rotate/scale block of an
  `LMatrix4`** from its translation row — this is the standard way to build
  an `LMatrix4` from a computed `LMatrix3` plus a separately-known
  translation (`LMatrix4(const LMatrix3&, const LVecBase3&)` constructor
  does exactly this), and how `invert_affine_from()` avoids doing 4x4 work.
- **`ident_mat()`/`y_to_z_up_mat()`/`z_to_y_up_mat()`/`ones_mat()`/`zeros_mat()`
  return references to cached static instances**, not fresh temporaries —
  cheap to call repeatedly. `LMatrix3`/`LMatrix4::convert_mat(from, to)`
  likewise returns a cached matrix converting between any pair of
  [CoordinateSystem](CoordinateSystem.md) values (composed from the same
  `_y_to_z_up_mat`/`_flip_y_mat`/`_flip_z_mat`/etc. private statics both
  classes carry).
- **`compare_to(other, threshold)` and `almost_equal()`/`generate_hash(..., threshold)`
  are out-of-line (`.cxx`), unlike almost everything else in `linmath`** —
  the threshold-taking matrix comparisons are large enough (9 or 16
  componentwise comparisons) that they aren't declared `INLINE_LINMATH`, so
  they don't get inlined into every call site the way the vector class
  methods do.

## API

Shape is nearly identical between `LMatrix3` and `LMatrix4`; differences
called out per-row. `Row`/`CRow` are internal helper classes supporting
`m[row][col]` two-level indexing — prefer `operator()(row, col)` or
`get_cell`/`set_cell` in agent-generated code for clarity.

### Construction / access
| Signature | Notes |
|---|---|
| `LMatrix3()` / `LMatrix4()` | Uninitialized |
| `LMatrix3(f,f,f, f,f,f, f,f,f)` / `LMatrix4(16 floats)` | Row-major element list |
| `LMatrix3(const LVecBase3&, const LVecBase3&, const LVecBase3&)` / `LMatrix4(4x LVecBase4)` | From rows |
| `LMatrix4(const LMatrix3 &upper3)` / `LMatrix4(const LMatrix3&, const LVecBase3 &trans)` | Embed a 3x3 into a 4x4, with optional translation |
| `void fill(FLOATTYPE)` / `set(...)` | |
| `FLOATTYPE operator()(int row, int col) const/&` | Preferred direct indexing |
| `Row/CRow operator[](int) const/&` | Two-level `m[row][col]` indexing |
| `void set_row/set_col(int, const LVecBase&)` / `LVecBase get_row/get_col(int) const` | `LMatrix4` also has `get_row3`/`get_col3`/`set_row`/`set_col` overloads taking `LVecBase3` for the 3-component sub-row |
| `void set_upper_3(const LMatrix3&)` / `LMatrix3 get_upper_3() const` | `LMatrix4` only |
| `const FLOATTYPE *get_data() const` / `iterator begin()/end()` | |
| `bool is_nan() const` / `bool is_identity() const` | `is_identity()` is `almost_equal(ident_mat(), NEARLY_ZERO)` |

### Transform application
| Signature | Notes |
|---|---|
| `LVecBase3 xform(const LVecBase3&) const` (`LMatrix3`) / `LVecBase4 xform(const LVecBase4&) const` (`LMatrix4`) | Fully general N-component transform |
| `LVecBase3 xform_point(const LVecBase3&) const` (`LMatrix4`) | Assumes affine; applies translation |
| `LVecBase3 xform_point_general(const LVecBase3&) const` (`LMatrix4`) | Handles projective matrices (perspective divide) |
| `LVecBase3 xform_vec(const LVecBase3&) const` (`LMatrix4`) / `LVecBase2/3 xform_vec` (`LMatrix3`) | Assumes affine/orthonormal; ignores translation |
| `LVecBase3 xform_vec_general(const LVecBase3&) const` | Uses inverse-transpose of the rotation block — correct under non-uniform scale/shear |
| `xform_in_place` / `xform_point_in_place` / `xform_vec_in_place` / `*_general_in_place` | In-place variants of all of the above |
| `operator*(const LVecBase3&, const LMatrix3&)` etc. (`lmat_ops_src.h`) | Free-function `v * M` spelling, separate overloads per LPoint/LVector/LVecBase and per arity |

### Matrix algebra
| Signature | Notes |
|---|---|
| `void multiply(const LMatrix&, const LMatrix&)` | `*this = other1 * other2` without an intermediate temporary |
| `operator* (const LMatrix&) const` / `*=` | Matrix product |
| `operator* (FLOATTYPE) const` / `operator/(FLOATTYPE) const` / `*=` / `/=` | Scalar scale |
| `operator+=` / `operator-=` | Componentwise |
| `void componentwise_mult(const LMatrix&)` | Elementwise, not matrix product |
| `FLOATTYPE determinant() const` | `LMatrix3` only |
| `void transpose_from(const LMatrix&)` / `transpose_in_place()` | |
| `bool invert_from(const LMatrix&)` / `invert_in_place()` | Returns `false` + resets to identity on singular; see behavior notes |
| `bool invert_affine_from(const LMatrix4&)` | `LMatrix4` only — fast path assuming no projective component |
| `bool invert_transpose_from(const LMatrix3&) const` / `invert_transpose_from(const LMatrix4&) const` | `LMatrix3` only — used by `xform_vec_general` |
| `void accumulate(const LMatrix4&, FLOATTYPE weight)` | `LMatrix4` only — `*this += other * weight`, an animation-blend primitive |

### Named-constructor factories (all take `CoordinateSystem cs = CS_default` where relevant)
| Signature | Notes |
|---|---|
| `set_translate_mat` / `translate_mat` (static) | `LMatrix3` takes `LVecBase2` (2-D translate); `LMatrix4` takes `LVecBase3` |
| `set_rotate_mat` / `rotate_mat` (static) | `LMatrix3` 2-D form takes just an angle; both classes' 3-D form takes `(angle, axis, cs)` |
| `set_rotate_mat_normaxis` / `rotate_mat_normaxis` (static) | Skips axis normalization — see behavior notes |
| `set_scale_mat` / `scale_mat` (static) | Uniform (`LMatrix4` only, single-float overload), per-axis, or `LVecBase2`/`LVecBase3` |
| `set_shear_mat` / `shear_mat` (static) | Per-axis shear values or an `LVecBase3` |
| `set_scale_shear_mat` / `scale_shear_mat` (static) | Combined scale+shear in one matrix |
| `static const LMatrix &ident_mat()` | `LMatrix4` also has `ones_mat()`/`zeros_mat()` |
| `static const LMatrix4 &y_to_z_up_mat() / z_to_y_up_mat()` | `LMatrix4` only |
| `static const LMatrix &convert_mat(CoordinateSystem from, CoordinateSystem to)` | Cached conversion matrix between any two coordinate systems |

### Comparison / hashing / I/O
| Signature | Notes |
|---|---|
| `bool operator<` / `operator==` / `operator!=` / `int compare_to(const LMatrix&) const` | Same lexicographic-total-order pattern as [LVecBase](LVecBase.md) |
| `int compare_to(const LMatrix&, FLOATTYPE threshold) const` | Out-of-line (`.cxx`) — see behavior notes |
| `size_t get_hash()/add_hash()` (with/without threshold) | |
| `bool almost_equal(const LMatrix&, FLOATTYPE threshold) const` / `almost_equal(const LMatrix&) const` | |
| `void output(std::ostream&) const` / `write(std::ostream&, int indent_level = 0) const` | `write()` pretty-prints one row per line, indented |
| `write_datagram_fixed` / `read_datagram_fixed` / `write_datagram` / `read_datagram` | Same fixed-vs-stdfloat distinction as [LVecBase](LVecBase.md) |

### Free functions
| Signature | Notes |
|---|---|
| `LMatrix transpose(const LMatrix&)` / `LMatrix invert(const LMatrix&)` | Non-mutating wrappers (declared `BEGIN_PUBLISH`/`END_PUBLISH` for scripting-language exposure) |
| `generic_write_datagram` / `generic_read_datagram` (`lmat_ops_src.h`) | Used by templated bam code |

## Usage

```cpp
LMatrix4 t = LMatrix4::translate_mat(LVector3(0, 5, 0));
LMatrix4 r = LMatrix4::rotate_mat(90, LVector3::up(CS_zup_right), CS_zup_right);
LMatrix4 world_xform = r * t;               // rotate, then translate (row-vector convention)

LPoint3 p(1, 0, 0);
LPoint3 world_p = p * world_xform;          // xform_point semantics: translation applies

LMatrix4 inv;
if (!inv.invert_from(world_xform)) {
  // world_xform was singular; inv is now the identity matrix
}
```

## See also

[LVecBase.md](LVecBase.md) / [LPoint.md](LPoint.md) / [LVector.md](LVector.md)
(the operand types) · [LQuaternion.md](LQuaternion.md) (`extract_to_matrix()`/
`set_from_matrix()` convert between quaternion and matrix rotation
representations) · [ComposeMatrix.md](ComposeMatrix.md) (build/decompose an
`LMatrix4` from scale/shear/hpr/translate) ·
[CoordinateSystem.md](CoordinateSystem.md) (`cs` parameters throughout) ·
[TransformState.md](../pgraph/TransformState.md) (caches the composed
`LMatrix4` for a scene-graph node) · [README.md](README.md)
