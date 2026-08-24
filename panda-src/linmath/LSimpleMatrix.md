# LSimpleMatrix

**Source:** `panda/src/linmath/lsimpleMatrix.h` / `.I`
**Inherits:** none (plain C++ class template)
**Inherited by:** none directly — it's a storage backend selected by macro, not a base class

`LSimpleMatrix<FloatType, NumRows, NumCols>` is the odd one out in
`linmath`: it's a **real C++ template**, not part of the
`fltnames.h`/`dblnames.h` macro-instantiation family the rest of the module
uses (see the [module README](README.md)). It exists as the fallback
row/column storage for every other `linmath` class
([LVecBase2/3/4](LVecBase.md), [LMatrix3/4](LMatrix.md)) when Panda is built
**without** the Eigen linear-algebra library — Eigen is used when available
because it brings SIMD-vectorized (SSE2/AVX) arithmetic; `LSimpleMatrix` is
a plain nested-array fallback with no vectorization, used only when Eigen
isn't compiled in.

## Behavior notes

- **Selection between `LSimpleMatrix` and `Eigen::Matrix` happens entirely
  through the `UNALIGNED_LINMATH_MATRIX`/`LINMATH_MATRIX` macros**, defined
  at the bottom of `lsimpleMatrix.h` right after the class itself: if
  `HAVE_EIGEN` is defined, both macros expand to `Eigen::Matrix<...>`
  (row-major, `DontAlign` for the unaligned form); otherwise both expand to
  `LSimpleMatrix<FloatType, NumRows, NumCols>`. Every `_v`/`_m` member across
  `LVecBase2`/`LVecBase3`/`LVecBase4`/`LMatrix3`/`LMatrix4` is typedef'd
  through one of these two macros — application code essentially never names
  `LSimpleMatrix` directly.
- **No arithmetic, no bounds checking, no anything beyond storage.** The
  entire class is `operator()(row, col)` (2-arg) and `operator()(col)`
  (1-arg, implicitly row 0 — used for the vector types' `_v(i)` accesses) —
  plain indexing into a `FloatType _array[NumRows][NumCols]`. All the actual
  math (`dot`, `cross`, matrix multiply, ...) each `linmath` class implements
  itself by looping over these accessors (see the `#ifdef HAVE_EIGEN /
  #else` branches throughout [LVecBase.md](LVecBase.md)'s and
  [LMatrix.md](LMatrix.md)'s source — every arithmetic method has a hand-rolled
  non-Eigen fallback that goes through this same indexing).
- **`ALIGN_LINMATH` (used by `LVecBase4`/`LMatrix4` for SSE2 alignment) is
  defined here too**, at the bottom of `lsimpleMatrix.h`, and depends on
  three things being simultaneously true: `LINMATH_ALIGN` defined, `HAVE_EIGEN`
  defined, and either `__AVX__` + `STDFLOAT_DOUBLE` (→ 32-byte alignment) or
  just `LINMATH_ALIGN` alone (→ 16-byte alignment) — when `HAVE_EIGEN` is
  *not* defined (i.e. `LSimpleMatrix` is actually in use), `ALIGN_LINMATH`
  expands to nothing, since there's no SIMD benefit to align for.

## API

| Signature | Notes |
|---|---|
| `const FloatType &operator()(int row, int col) const` / `FloatType &operator()(int row, int col)` | Direct 2-D indexing into `_array` |
| `const FloatType &operator()(int col) const` / `FloatType &operator()(int col)` | 1-D indexing — row 0 only, used by the vector classes |

## Usage

Not meant to be used directly by application or agent-generated code — it's
selected transparently as `LVecBase3::_v`'s or `LMatrix4::_m`'s type when
Eigen isn't compiled in. If you're writing code that touches `_v`/`_m`
directly (rare — normally you go through the public accessors on
[LVecBase](LVecBase.md)/[LMatrix](LMatrix.md)), it works identically whether
the underlying type is `LSimpleMatrix` or `Eigen::Matrix`, since both expose
the same `operator()(row, col)` shape.

## See also

[LVecBase.md](LVecBase.md) / [LMatrix.md](LMatrix.md) (the classes that
actually use this as backing storage) · [README.md](README.md)
