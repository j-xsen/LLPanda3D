# LPlane

**Source:** `panda/src/mathutil/plane_src.h/.I/.cxx` (macro-templated body) +
`plane.h` (float/double instantiation wrapper)
**Inherits:** `FLOATNAME(LVecBase4)` — i.e. `LVecBase4f`/`LVecBase4d` from
`linmath` (see [../linmath/README.md](../linmath/README.md))
**Inherited by:** (none)

A plane in the form `Ax + By + Cz + D = 0`, stored as an `LVecBase4`
(`(A, B, C, D)` — the first three components are the normal, `D` is the
signed offset). Used throughout the engine for view-frustum face planes
([BoundingHexahedron](BoundingHexahedron.md)/[Frustum](Frustum.md)),
half-space bounds ([BoundingPlane](BoundingPlane.md)), and
[collide](../collide/CollisionPlane.md)'s `CollisionPlane`/`CollisionPolygon`.

**This file follows the same `fltnames.h`/`dblnames.h` macro-instantiation
pattern as `linmath`'s own vector/matrix types**: `plane_src.h` is written
once using `FLOATNAME(LPlane)`/`FLOATTYPE`/`FLOATCONST` macros, then
`plane.h` `#include`s it twice — once after `#include "fltnames.h"` (binds
`FLOATNAME(X)` → `Xf`, `FLOATTYPE` → `PN_float32`, etc.) to produce
`LPlanef`, and once after `#include "dblnames.h"` to produce `LPlaned`.
`LPlane` itself is then a `typedef` to whichever matches the build's
`STDFLOAT_DOUBLE` setting. See [../linmath/README.md](../linmath/README.md)
for the full explanation of this code-generation mechanism — it isn't
re-explained per file in this module.

## Behavior notes

- **`dist_to_plane(point)` is signed**: positive means the point is in
  front of the plane (the side the normal points toward), negative means
  behind, zero means exactly on the plane. Nearly every other method in this
  class is built on this one primitive.
- **The 3-point constructor expects counter-clockwise winding as viewed from
  in front of the plane** (i.e. from the end of the normal looking back
  toward the plane) — `normal = normalize(cross(b - a, c - a))`. Reversing
  the winding flips the normal, and therefore flips what "in front"/"behind"
  means for every subsequent test.
- **`normalize()`/`normalized()` treat a zero-length normal as a no-op
  failure, not an error**: `normalize()` returns `false` and leaves the
  plane untouched if `get_normal().length_squared() == 0`; `normalized()`
  likewise just returns a copy of the original plane in that case. Also
  note `normalize()` skips the actual division (leaving values as-is) when
  the normal is already extremely close to unit length
  (`IS_THRESHOLD_EQUAL` against `NEARLY_ZERO(FLOATTYPE)^2`), a small
  optimization to avoid an unnecessary divide-and-roundoff on an
  already-normalized plane.
- **`operator *` (transform by matrix) has two overloads with different
  correctness properties.** The `LMatrix3` overload transforms the normal
  directly (`mat.xform(normal)`) and re-derives `D` from the *original*
  point via `get_point()` — appropriate for pure-rotation matrices. The
  `LMatrix4` overload instead uses `xform_vec_general()` for the normal
  (the mathematically correct "transform by the inverse-transpose" approach
  for non-uniform scale/shear, not just a plain vector transform) and
  transforms `get_point()` by the *forward* matrix for the new point — see
  `../linmath` for what `xform_vec_general()` does differently from
  `xform_vec()`. Using the wrong overload (or a plain vector transform) on
  a plane under non-uniform scale silently produces a wrong plane.
- **`get_point()` picks whichever axis has the largest-magnitude normal
  component to avoid dividing by a near-zero component** — not an arbitrary
  point in any fixed sense; it returns a different point depending on the
  plane's orientation, subject to `nassertr(component != 0)` on the chosen
  axis (only fails if the plane is completely degenerate, i.e. its normal
  is exactly zero along all three axes at once).
- **`intersects_line(t, from, delta)` returns `false` (not an error) when
  the line is parallel to the plane** (`dot(normal, delta)` nearly zero) —
  the "point + line direction" overload then derives the actual
  intersection point from `t`, so a parallel/no-intersection line simply
  produces no output rather than a garbage point.
- **`intersects_plane()` solves for the line of two planes' intersection
  using the standard closed-form (cross product for direction, then a 2x2
  linear solve in the `(n1, n2)` basis for a point on the line)** — returns
  `false` only when the cross product is (nearly) zero, i.e. the planes are
  parallel. Note it does *not* check whether the planes are also
  coincident (same plane) vs. merely parallel-and-non-intersecting; both
  cases return `false` since a coincident pair has no unique line of
  intersection either.
- **`intersects_parabola()` substitutes the parabola's parametric form
  directly into the plane equation to get a quadratic in `t`** (see
  [Parabola.md](Parabola.md)) — handles the degenerate a≈0 (parabola's
  quadratic term has no component along the normal, reduces to linear) and
  a≈0∧b≈0 (parabola is entirely parallel to or embedded in the plane;
  treated as "no intersection" either way, even the embedded case) cases
  explicitly before falling to the general quadratic formula.
- **`get_reflection_mat()` builds a Householder-style reflection matrix**
  directly from the plane's `(A,B,C,D)` coefficients (`I - 2*n*n^T` for the
  rotation part, plus a translation term from `D`) — used for mirror/portal
  rendering effects.
- `write_datagram_fixed()`/`read_datagram_fixed()` (fixed-width float32/64,
  independent of `Datagram::set_stdfloat_double()`) vs.
  `write_datagram()`/`read_datagram()` (uses the standard-width setting) —
  the same fixed-vs-standard pattern used by other linmath-adjacent types;
  see [Parabola.md](Parabola.md) for the same distinction spelled out.

## API

### Construction
| Signature | Notes |
|---|---|
| `LPlane()` | Default: Z=1 plane through the origin |
| `LPlane(const LVecBase4 &copy)` | Reinterprets a 4-vector as `(A,B,C,D)` |
| `LPlane(const LPoint3 &a, const LPoint3 &b, const LPoint3 &c)` | CCW winding as seen from in front — see notes |
| `LPlane(const LVector3 &normal, const LPoint3 &point)` | Normal + a point in the plane |
| `LPlane(FLOATTYPE a, FLOATTYPE b, FLOATTYPE c, FLOATTYPE d)` | Raw coefficients |

### Queries
| Signature | Notes |
|---|---|
| `LVector3 get_normal() const` | First 3 components |
| `LPoint3 get_point() const` | See notes — axis chosen to avoid near-zero division |
| `FLOATTYPE dist_to_plane(const LPoint3 &point) const` | Signed distance |
| `LPoint3 project(const LPoint3 &point) const` | Nearest point in the plane |
| `bool normalize()` / `LPlane normalized() const` | See notes re: zero-normal handling |
| `void flip()` | Negates all four coefficients (flips which side is "front") |
| `LPlane operator - () const` | Same as `flip()` but returns a copy |

### Transform
| Signature | Notes |
|---|---|
| `LPlane operator * (const LMatrix3 &mat) const` | Rotation-only transform — see notes |
| `LPlane operator * (const LMatrix4 &mat) const` / `operator *= ` / `void xform(const LMatrix4&)` | General transform via `xform_vec_general()` — see notes |
| `LMatrix4 get_reflection_mat() const` | Mirror-reflection matrix across the plane |

### Intersection
| Signature | Notes |
|---|---|
| `bool intersects_line(LPoint3 &intersection_point, const LPoint3 &p1, const LPoint3 &p2) const` | Line through two points |
| `bool intersects_line(FLOATTYPE &t, const LPoint3 &from, const LVector3 &delta) const` | Parametric form; `false` if parallel |
| `bool intersects_plane(LPoint3 &from, LVector3 &delta, const LPlane &other) const` | Line of intersection of two planes |
| `bool intersects_parabola(FLOATTYPE &t1, FLOATTYPE &t2, const LParabola &parabola) const` | See [Parabola.md](Parabola.md) |

### I/O
| Signature | Notes |
|---|---|
| `void output(std::ostream&) const` / `write(std::ostream&, int = 0) const` / `std::string __repr__() const` | |

## Usage

```cpp
LPlane ground(LVector3(0, 0, 1), LPoint3(0, 0, 0));
LPoint3 hit;
if (ground.intersects_line(hit, ray_start, ray_start + ray_dir * 1000.0)) {
  // hit is where the ray crosses the ground plane
}
```

## See also

[BoundingPlane.md](BoundingPlane.md) · [Frustum.md](Frustum.md) ·
[Parabola.md](Parabola.md) · [../linmath/README.md](../linmath/README.md) ·
[../collide/CollisionPlane.md](../collide/CollisionPlane.md) · [README.md](README.md)
