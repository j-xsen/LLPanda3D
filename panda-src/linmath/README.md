# Linmath — Vectors, Points, Matrices, Quaternions

**Source:** `panda/src/linmath/` · Library: `libp3linmath` · Notify
category: `linmath`

`linmath` is Panda3D's linear-algebra core: the vector/point/matrix/quaternion
types used by every other module that touches 3-D space — scene graph
transforms ([TransformState](../pgraph/TransformState.md)), collision solids
([CollisionSolid](../collide/CollisionSolid.md) and its subclasses),
animation channels ([AnimChannelMatrix](../chan/AnimChannelMatrix.md)),
lenses/projection, and more. It has no dependency on rendering or the scene
graph itself — it's pure math plus a config-driven notion of which way is
"up."

## The macro-instantiation mechanism (read this first)

Most of this module isn't hand-written per numeric type. A single
`*_src.h`/`*_src.I`/`*_src.cxx` triple (e.g. `lvecBase3_src.h`) holds one
real implementation, written in terms of placeholder macros
(`FLOATTYPE`, `FLOATNAME(x)`, `FLOATCONST(x)`, ...). The **outer** header
(`lvecBase3.h`) `#include`s a *name-header* immediately before each
`#include` of the `_src.h` file, and the name-header (re)defines those
placeholder macros for one concrete numeric type:

```cpp
#include "fltnames.h"        // FLOATTYPE=float,  FLOATNAME(x)=x##f  -> LVecBase3f
#include "lvecBase3_src.h"

#include "dblnames.h"        // FLOATTYPE=double, FLOATNAME(x)=x##d  -> LVecBase3d
#include "lvecBase3_src.h"

#include "intnames.h"        // FLOATTYPE=int,    FLOATNAME(x)=x##i  -> LVecBase3i
#include "lvecBase3_src.h"
```

The same `.h` text gets textually re-included and re-preprocessed three
times, each time under different macro definitions, producing three
distinct classes (`LVecBase3f`, `LVecBase3d`, `LVecBase3i`). The project's
own comment in `fltnames.h` calls this "a poor man's template... to avoid
some of the inherent problems with templates: compiler complexity and
distributed code bloat... plus if-based specialization... for compilers
(like VC++) that don't completely support template specialization." **Only
[LVecBase2/3/4](LVecBase.md), [LPoint2/3/4](LPoint.md), and
[LVector2/3/4](LVector.md)** (plus the free-function `lvec*_ops_src.h`
files) get the `intnames.h` treatment — everything else in the module
([LMatrix3/4](LMatrix.md), [LQuaternion](LQuaternion.md),
[LRotation/LOrientation](LRotation.md), [compose_matrix](ComposeMatrix.md))
is `float`/`double` only; rotation and matrix math on integers isn't
meaningful. Code inside a `_src.h`/`_src.I`/`_src.cxx` file frequently
branches on `#ifdef FLOATTYPE_IS_INT` to skip methods that don't make sense
for the integer instantiation (`length()`, `normalize()`, threshold-based
comparisons, ...).

`f`/`d` (and `i` where present) are always both compiled in — the
unqualified names (`LVecBase3`, `LPoint3`, `LMatrix4`, `LQuaternion`, ...)
are **typedefs**, defined in `aa_luse.h` (included via the umbrella header
`luse.h`), that resolve to the `f` or `d` version depending on whether
`STDFLOAT_DOUBLE` was defined at build time — the default is single
precision. `luse.h`/`aa_luse.h`/`luse.cxx`/`luse.I` also define a handful of
semantic aliases on top of these: `LVertex`/`LNormal`/`LTexCoord`/`LColor`/
`LRGBColor` (each `LPoint3`/`LVector3`/`LPoint2`/`LVecBase4`/`LVecBase3`
respectively, `f`/`d` per `STDFLOAT_DOUBLE`) — agents reading Panda code
that uses `LColor` or `LVertex` are looking at these aliases, not distinct
classes. There's no separate doc page for `luse.h` beyond this section; it
has no API of its own besides re-exporting.

Two more name-header pairs exist purely to support **cross-type**
conversion: `flt2dblnames.h`/`dbl2fltnames.h` (used only by
[lcast_to](LCastTo.md)) define **both** the primary `FLOATTYPE`/`FLOATNAME`
*and* a second `FLOATTYPE2`/`FLOATNAME2` for "the other" numeric type, so a
single `_src.h` body can declare both directions of a float↔double
conversion function at once.

## Class map

**Coordinate-tuple family (float/double/int, built on shared storage):**
```
LVecBase2 / LVecBase3 / LVecBase4        (LVecBase.md)  — raw N-tuples, no point/vector semantics
UnalignedLVecBase4                       (LVecBase.md)  — tightly-packed, non-SSE2-aligned LVecBase4
├── LPoint2 / LPoint3 / LPoint4          (LPoint.md)    — a location in space
└── LVector2 / LVector3 / LVector4       (LVector.md)   — a direction/displacement
    └── LQuaternion (: LVecBase4)        (LQuaternion.md) — unit quaternion, rotation
        ├── LRotation                    (LRotation.md) — semantic: "a rotation to apply"
        └── LOrientation                 (LRotation.md) — semantic: "an absolute facing"
```

**Matrices:**
```
LMatrix3 / LMatrix4                      (LMatrix.md)   — row-major transform matrices
UnalignedLMatrix4                        (LMatrix.md)   — tightly-packed, non-SSE2-aligned LMatrix4
LSimpleMatrix<FloatType, R, C>           (LSimpleMatrix.md) — real C++ template; non-Eigen storage fallback for all of the above
```

**Composition / conversion / config (free functions, no classes):**
```
compose_matrix() / decompose_matrix()    (ComposeMatrix.md) — LMatrix <-> scale/shear/hpr/translate
CoordinateSystem (enum) + helpers        (CoordinateSystem.md) — Z-up/Y-up, left/right-handed
deg_2_rad() / rad_2_deg() / MathNumbers  (MathConstants.md) — angle conversion, pi/ln2 constants
lcast_to() / cast_to_double() / cast_to_float()  (LCastTo.md) — float<->double (and int->float/double) conversion
ConfigVariableColor (: ConfigVariable)   (ConfigVariableColor.md) — prc-config-backed LColor value
```

## Not documented here

- **`*_ext.h`/`*_ext_src.*`** (`lvecBase2/3/4_ext_src`, `lpoint2/3/4_ext_src`,
  `lvector2/3/4_ext_src`, `lmatrix3/4_ext_src`) — Python `EXTENSION(...)`
  method implementations (`__reduce__`, `__repr__`, `__floordiv__`,
  `__pow__`, `__round__`/`__floor__`/`__ceil__`). One behavior worth
  flagging even though it's Python-only: `__getattr__`/`__setattr__` on the
  `LVecBase*_ext_src` classes implement Python-side vector **swizzling**
  (`v.xz`, `v.yyx`, ...) by validating each character of the requested
  attribute name against the class's axis letters and building a
  same-or-smaller vector — noted in [LVecBase.md](LVecBase.md)'s behavior
  notes since agents may encounter these attribute names when translating
  Python Panda3D code, even though there's no C++ equivalent.
- **`config_linmath.h/.cxx`** — module init boilerplate (`init_liblinmath()`
  registers every `f`/`d`/`i` class's `TypeHandle`) plus the notify category
  (`linmath`, noted above) and the `paranoid-hpr-quat` / `no-singular-invert`
  `ConfigVariableBool`s, both mentioned in [LQuaternion.md](LQuaternion.md)
  and [LMatrix.md](LMatrix.md) respectively where they're actually consumed.
- **`p3linmath_composite1.cxx`, `p3linmath_composite2.cxx`** — build-system
  unity-build wrapper files, not real source.
- **`test_math.cxx`** — standalone test/demo program, not library code.
- **`intnames.h`, `fltnames.h`, `dblnames.h`, `flt2dblnames.h`,
  `dbl2fltnames.h`** — the macro-naming headers themselves; the mechanism is
  explained above rather than documented as classes (there's nothing to
  document — they're `#define` lists with no runtime behavior).
- **`luse.h`/`.I`/`.cxx`/`aa_luse.h`** — the umbrella include and its
  `LVertex`/`LNormal`/`LColor`/etc. typedef aliases; covered in the macro
  mechanism section above, not a separate page. `luse.cxx` is a single
  `#include "luse.h"` with no code of its own.
- **`luse.N`** — an `interrogate` (Panda's C++→Python binding generator)
  directive file, not C++: `defconstruct` lines tell the binding generator
  what "default-construct" means for each `f`/`d`/`i` class (e.g.
  `LMatrix4f` defaults to `ident_mat()`, `LQuaternionf` to `ident_quat()`,
  everything else to a zero-fill), and `noinclude` lines tell it not to
  independently parse the `_src.h` files as top-level headers, since they
  only produce valid code once wrapped by a name-header.

## Core concepts

**A single `_src.h` file becomes two or three real classes via textual
re-inclusion under different macro definitions, not templates.** See "The
macro-instantiation mechanism" above — this is the one thing every doc page
in this module assumes you already understand, since every behavior note
that says "non-`i` only" or "`f`/`d` only" is talking about which
name-headers a given `_src.h` gets `#include`d under.

**Point vs. vector is a type-system distinction with zero storage cost.**
[LPoint](LPoint.md) and [LVector](LVector.md) add no fields over
[LVecBase](LVecBase.md) — same `_v` layout, same size, sometimes even
literally reinterpret-cast between each other's static constants
(`LVector3::zero()` casts `LVecBase3::zero()`'s reference). The entire
payoff is compile-time-enforced arithmetic rules (`point - point = vector`,
`point + vector = point`, mixed operations demote to the vecbase type) and,
critically, different behavior under [LMatrix](LMatrix.md) transforms:
`xform_point()` applies translation, `xform_vec()` doesn't. Reach for
`LVecBase` directly only when neither point nor vector applies (a plane
equation, a color).

**Vectors are row vectors; matrices multiply on the right.** `v * M`, not
`M * v` — this is why a matrix's translation lives in row 3 (the fourth
row), and why chained transforms read left-to-right in application order
(`local_to_world = local_to_parent * parent_to_world`). See
[LMatrix.md](LMatrix.md).

**"Up" is a config variable, not a hard-coded axis.** Every function that
takes `CoordinateSystem cs = CS_default` — `LVector3::up/right/forward()`,
`LMatrix::set_rotate_mat()`, `compose_matrix()`, `LQuaternion::set_hpr()` —
resolves `CS_default` through `get_default_coordinate_system()`, which reads
the `coordinate-system` prc variable (default `CS_zup_right`). The same
logical rotation produces a different concrete matrix depending on whether
the active system is Z-up or Y-up, right- or left-handed — see
[CoordinateSystem.md](CoordinateSystem.md) and the axis-mapping table in
[LVector.md](LVector.md).

**Eigen is an optional accelerant, not a hard dependency.** Every arithmetic
method in [LVecBase](LVecBase.md)/[LMatrix](LMatrix.md) has two
implementations side by side (`#ifdef HAVE_EIGEN ... #else ... #endif`) — the
Eigen path uses `Eigen::Matrix` for SIMD-accelerated math, the fallback path
hand-loops over [LSimpleMatrix](LSimpleMatrix.md), a genuine (non-macro) C++
template that's otherwise invisible to application code. Which one backs
`_v`/`_m` is chosen entirely by the `UNALIGNED_LINMATH_MATRIX`/
`LINMATH_MATRIX` macros at the bottom of `lsimpleMatrix.h`.

**Quaternion-vs-matrix-vs-hpr rotation math is cross-checked at runtime, not
just tested once.** `LQuaternion::set_hpr()`/`get_hpr()`, when the
`paranoid-hpr-quat` prc variable is set (debug builds only), independently
recompute the same result via [compose_matrix](ComposeMatrix.md)/
`decompose_matrix()` and log a warning (overwriting the result with the
matrix-derived answer) if the two disagree — see
[LQuaternion.md](LQuaternion.md).

## File index

| Topic | Purpose |
|---|---|
| [LVecBase.md](LVecBase.md) | `LVecBase2/3/4` — raw coordinate tuples (`f`/`d`/`i`), the storage base for everything below |
| [LPoint.md](LPoint.md) | `LPoint2/3/4` — a location in space; point/vector arithmetic rules |
| [LVector.md](LVector.md) | `LVector2/3/4` — a direction/displacement; `up`/`right`/`forward`/`rfu`, angle-between |
| [LMatrix.md](LMatrix.md) | `LMatrix3/4` + `UnalignedLMatrix4` — row-major transform matrices, `xform_point`/`xform_vec` |
| [LQuaternion.md](LQuaternion.md) | Unit quaternion rotation; hpr/matrix conversion, `is_same_direction` |
| [LRotation.md](LRotation.md) | `LRotation` / `LOrientation` — semantic quaternion subclasses (delta vs. absolute facing) |
| [LSimpleMatrix.md](LSimpleMatrix.md) | Non-Eigen fallback storage template used by every class above |
| [ComposeMatrix.md](ComposeMatrix.md) | `compose_matrix()`/`decompose_matrix()` — matrix ↔ scale/shear/hpr/translate |
| [CoordinateSystem.md](CoordinateSystem.md) | `CoordinateSystem` enum, `get_default_coordinate_system()`, handedness |
| [MathConstants.md](MathConstants.md) | `deg_2_rad()`/`rad_2_deg()`, `MathNumbers` (pi, ln2) |
| [ConfigVariableColor.md](ConfigVariableColor.md) | prc-config-backed `LColor` value |
| [LCastTo.md](LCastTo.md) | `lcast_to()`/`LCAST` macro, `cast_to_double()`/`cast_to_float()` — precision conversion |

## Status

linmath — done (2026-08-24). See [../../README.md](../../README.md) for the
overall index across `panda/src/*` modules.
