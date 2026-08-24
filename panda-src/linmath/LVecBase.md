# LVecBase2 / LVecBase3 / LVecBase4

**Source:** `panda/src/linmath/lvecBase2_src.h/.I` (`.cxx`), `lvecBase3_src.*`,
`lvecBase4_src.*` — each `#include`d three times (once per numeric type) by
`lvecBase2.h`/`lvecBase3.h`/`lvecBase4.h` · plus `lvecBase2_ext_src.*` /
`lvecBase3_ext_src.*` / `lvecBase4_ext_src.*` (Python interop) and
`lvec2_ops_src.*` / `lvec3_ops_src.*` / `lvec4_ops_src.*` (free-function
operators) · Library: `libp3linmath` · Notify category: `linmath`
**Inherits:** none (value types; register a `TypeHandle` but don't derive
from `TypedObject`)
**Inherited by:** [LPoint2/3/4](LPoint.md), [LVector2/3/4](LVector.md)
(direct); [LQuaternion](LQuaternion.md) inherits `LVecBase4`

`LVecBase2`/`LVecBase3`/`LVecBase4` are the raw N-component numeric tuples
that everything else in `linmath` builds on: a flat array of `FLOATTYPE`
with componentwise arithmetic, no semantic meaning attached. Use them
directly only when neither "point" nor "vector" applies — e.g. a plane
equation, a texture-matrix row, or (via the `LColor`/`LRGBColor` typedefs) a
color value. Otherwise reach for [LPoint](LPoint.md) or
[LVector](LVector.md), which add the point/vector distinction on top of the
identical storage layout. See the [module README](README.md) for how the
single `*_src.h` source file becomes three real classes (`f`/`d`/`i` suffix)
via the `fltnames.h`/`dblnames.h`/`intnames.h` macro-naming headers.

## Behavior notes

- **Every instantiation is generated three times, not two.** Unlike most of
  `linmath` (which is `float`/`double` only), `LVecBase2`/`3`/`4` (and their
  [LPoint](LPoint.md)/[LVector](LVector.md) subclasses, and the
  `lvec*_ops_src` free functions) are also instantiated under `intnames.h`,
  producing `LVecBase3i` etc. — see `lvecBase3.h`'s three `#include
  "lvecBase3_src.h"` lines, one per name-header. Integer instantiations skip
  every method gated `#ifndef FLOATTYPE_IS_INT` — no `length()`,
  `normalize()`, `normalized()`, `project()`, no threshold-based
  `compare_to()`/`get_hash()`/`generate_hash()` overloads (integers compare
  exactly, no epsilon needed), and no `get_standardized_hpr()` (LVecBase3
  only). `enum { is_int = 1 }` (vs. `0`) lets templated/generic code branch
  on which flavor it has at compile time.
- **Storage is a single Eigen row vector, or a hand-rolled fallback.** `_v`
  is `EVector3`/`EVector4`/etc., typedef'd through
  `UNALIGNED_LINMATH_MATRIX`/`LINMATH_MATRIX` (see [LSimpleMatrix](LSimpleMatrix.md))
  to either an `Eigen::Matrix<FLOATTYPE, 1, N, RowMajor>` when Eigen is
  available, or a plain `LSimpleMatrix<FLOATTYPE, 1, N>` otherwise. `LVecBase2`
  and `LVecBase3` are deliberately **not** aligned (`UNALIGNED_LINMATH_MATRIX`)
  — too few floats to benefit from SSE2 and not worth the alignment hassle —
  while `LVecBase4` **is** 16-byte aligned (`ALIGN_LINMATH` +
  `LINMATH_MATRIX`) specifically to enable SSE2 on the 4-float case. This is
  why `UnalignedLVecBase4` exists as a separate class (see below).
- **`UnalignedLVecBase4`/`FLOATNAME(UnalignedLVecBase4)`** (declared inside
  `lvecBase4_src.h`, not a separate file) is a bare-bones 4-float tuple with
  no methods beyond construction, `fill()`/`set()`, indexing, and
  `==`/`!=` — no arithmetic, no `TypeHandle`-driven virtual behavior beyond
  registration. It exists purely so code that needs to pack 4 floats tightly
  (no padding) can do so and then convert to a proper `LVecBase4` when it
  needs actual math. `LVecBase4` has a constructor taking an
  `UnalignedLVecBase4` and vice versa.
- **`operator[]`/`get_cell`/`set_cell` all `nassertr`/`nassertv`-assert the
  index is in range** (`0 <= i < num_components`) rather than doing anything
  defensive in release builds — out-of-range access on a release build is
  simply undefined (raw array indexing into `_v`).
- **`normalize()` (in place) treats "already unit length" as a no-op, not
  just "avoid division by zero."** It checks `length_squared()` against `1.0`
  within `NEARLY_ZERO(FLOATTYPE)²` and skips the divide entirely if already
  normalized — `normalize()`/`normalized()` on a zero vector sets everything
  to `0` and returns `false` (or returns a zero vector) rather than
  dividing by zero.
- **`cross()` only exists on `LVecBase3`** (2-component and 4-component cross
  products aren't defined here); `LVecBase3::cross_into()` is the in-place
  form (`*this = cross(other)`), and the standalone `cross()` free function
  in `lvec3_ops_src.h` is just a thin wrapper.
- **`compare_to()`/`get_hash()`/`generate_hash()` each come in two flavors**
  for non-integer types: a threshold-less default (uses
  `NEARLY_ZERO(FLOATTYPE)`, componentwise `IS_THRESHOLD_COMPEQ`) and an
  explicit-threshold overload. `operator<` is defined purely in terms of
  `compare_to()` and, like [BitMask](../putil/BitMask.md)'s, is a
  lexicographic total order for STL container use, not a subset/magnitude
  comparison.
- **`write_datagram_fixed()`/`read_datagram_fixed()` always use a fixed
  32-/64-bit width matching `FLOATTOKEN`** (`add_float32`/`add_float64`, or
  `add_int32` for the `i` variant), regardless of
  `Datagram::set_stdfloat_double()`; `write_datagram()`/`read_datagram()`
  instead use `add_stdfloat()`/`get_stdfloat()`, which respects that global
  setting — use `_fixed` when you need a portable, type-stable wire format
  and the plain form when writing an actual `.bam` file (see
  [DatagramFile.md](../putil/DatagramFile.md)).
- **`LVecBase3::get_xy()`/`get_xz()`/`get_yz()` and `LVecBase4::get_xyz()`/`get_xy()`
  slice down to a smaller `LVecBase`, never up** — there's no `LVecBase2`
  constructor that pads with zeros implicitly; going from `LVecBase2` to
  `LVecBase3` requires the explicit `(const LVecBase2 &copy, FLOATTYPE z)`
  constructor.
- **`lvec2_ops_src.h`/`lvec3_ops_src.h`/`lvec4_ops_src.h`** supply the
  free-function forms that don't fit as member operators: `scalar * vec`
  (member operators only cover `vec * scalar`), standalone `dot()`/`cross()`/
  `length()`/`normalize()` wrappers around the member methods, and
  `generic_write_datagram()`/`generic_read_datagram()` (used by templated bam
  code that doesn't know the concrete vector type at compile time).
- **Python `__getattr__`/`__setattr__` (in the `*_ext_src` extension
  classes) implement swizzling** — `v.xz`, `v.yyx`, etc. — by validating each
  character of the requested attribute name is in the class's own axis-letter
  range (`x`..`z` for `LVecBase3`, `x`..`w` for `LVecBase4`) and building a
  same-or-smaller `LVecBase` of the right arity; not part of the C++ API at
  all, mentioned only because agents translating Python Panda3D code will see
  these attribute names.

## API

### Construction / statics (shape is identical across LVecBase2/3/4; shown for LVecBase3)
| Signature | Notes |
|---|---|
| `LVecBase3()` | Uninitialized (`= default`) |
| `LVecBase3(FLOATTYPE fill_value)` | All components set to this value |
| `LVecBase3(FLOATTYPE x, FLOATTYPE y, FLOATTYPE z)` | |
| `LVecBase3(const LVecBase2 &copy, FLOATTYPE z)` | Widen by one component |
| `static const LVecBase3 &zero() / unit_x() / unit_y() / unit_z()` | Cached constants, not per-call allocations |
| `constexpr static int size()` / `get_num_components()` | `= 2`/`3`/`4` |
| `enum { num_components, is_int }` | `is_int` is `1` only for the `...i` instantiation |

### Element access
| Signature | Notes |
|---|---|
| `FLOATTYPE operator[](int i) const` / `FLOATTYPE &operator[](int i)` | Asserts `0 <= i < num_components` |
| `get_x/y/z/w()` / `set_x/y/z/w(FLOATTYPE)` | Also `MAKE_PROPERTY` (`.x`, `.y`, ...) for scripting languages |
| `get_cell(int)` / `set_cell(int, FLOATTYPE)` | Index form of the above |
| `add_x/y/z/w(FLOATTYPE)` / `add_to_cell(int, FLOATTYPE)` | `set_x(get_x() + value)` shorthand, avoids two calls from scripting languages |
| `LVecBase2 get_xy/xz/yz() const` (3→2) · `LVecBase3 get_xyz() const`, `LVecBase2 get_xy() const` (4→3/2) | Slice to a smaller vector; no widening equivalent |
| `const FLOATTYPE *get_data() const` | Pointer to first of N contiguous elements |
| `iterator begin()/end()` (`const FLOATTYPE*`) | STL-style traversal |
| `bool is_nan() const` | Always `false` for the `i` instantiation |

### Arithmetic
| Signature | Notes |
|---|---|
| `fill(FLOATTYPE)` / `set(...)` | |
| `dot(const LVecBase&) const` / `length_squared() const` | Both work for the `i` instantiation |
| `length() const` / `normalize()` / `normalized() const` / `project(const LVecBase&) const` | **Not** available on the `i` instantiation |
| `cross(const LVecBase3&) const` / `void cross_into(const LVecBase3&)` | `LVecBase3` only |
| `operator + - * / += -= *= /=` | `vec * scalar` and `vec / scalar` are members; `scalar * vec` is a free function in `lvec*_ops_src.h` |
| `void componentwise_mult(const LVecBase&)` | Elementwise multiply (no operator spelling; not the same as `dot()`) |
| `fmax(const LVecBase&) const` / `fmin(const LVecBase&) const` | Componentwise max/min |

### Comparison / hashing
| Signature | Notes |
|---|---|
| `bool operator==/!=` | Exact componentwise equality |
| `bool operator<` / `int compare_to(const LVecBase&) const` | Lexicographic, not "less in magnitude"; usable as an STL ordering key |
| `int compare_to(const LVecBase&, FLOATTYPE threshold) const` | Non-`i` only |
| `size_t get_hash() const` / `add_hash(size_t) const` / `generate_hash(ChecksumHashGenerator&) const` | Default and threshold-taking overloads (non-`i` only for the threshold forms) |
| `bool almost_equal(const LVecBase&, FLOATTYPE threshold) const` / `almost_equal(const LVecBase&) const` | Default threshold from `NEARLY_ZERO(FLOATTYPE)` |
| `LVecBase3 get_standardized_hpr() const` | `LVecBase3`, non-`i` only; wraps each component into `[-180, 180)` — see [LQuaternion.md](LQuaternion.md) for why this is unreliable near gimbal-lock and `is_same_direction()` is usually the better check |

### I/O
| Signature | Notes |
|---|---|
| `void output(std::ostream&) const` / `operator<<` | Space-separated components |
| `write_datagram_fixed(Datagram&) const` / `read_datagram_fixed(DatagramIterator&)` | Fixed 32-/64-bit width regardless of stdfloat setting |
| `write_datagram(Datagram&) const` / `read_datagram(DatagramIterator&)` | Uses `add_stdfloat()`/`get_stdfloat()` — appropriate for `.bam` files |

### Free functions (`lvec2/3/4_ops_src.h`)
| Signature | Notes |
|---|---|
| `LVecBase3 operator*(FLOATTYPE scalar, const LVecBase3 &a)` (+ LPoint3/LVector3 overloads) | `scalar * vec` form |
| `FLOATTYPE dot(const LVecBase3&, const LVecBase3&)` | Free-function wrapper |
| `LVecBase3 cross(const LVecBase3&, const LVecBase3&)` / `LVector3 cross(const LVector3&, const LVector3&)` | `LVecBase3`/`LVector3` overloads (2/4-component have no `cross`) |
| `FLOATTYPE length(const LVecBase3&)` / `LVector3 normalize(const LVecBase3&)` | Non-`i` only |
| `generic_write_datagram(Datagram&, const LVecBase3&)` / `generic_read_datagram(LVecBase3&, DatagramIterator&)` | Used by templated bam-writing code |

## Usage

```cpp
LVector3 axis(0, 0, 1);
LVector3 offset(1.0, 2.0, 0.5);

LVecBase3 combined = offset + axis * 3.0f;   // vec + scalar*vec
float d = dot(offset, axis);
LVector3 n = normalize(offset);              // free-function form

if (offset.almost_equal(LVecBase3(1.0, 2.0, 0.5), 0.001f)) {
  // within tolerance
}
```

## See also

[LPoint.md](LPoint.md) / [LVector.md](LVector.md) (semantic subclasses built
on this identical layout) · [LMatrix.md](LMatrix.md) (`vec * matrix`
transforms) · [LSimpleMatrix.md](LSimpleMatrix.md) (the non-Eigen storage
fallback) · [ComposeMatrix.md](ComposeMatrix.md) (scale/shear/hpr built from
`LVecBase3`s) · [README.md](README.md)
