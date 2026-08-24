# BoundingSphere

**Source:** `panda/src/mathutil/boundingSphere.h` (+ `.I`, `.cxx`)
**Inherits:** `FiniteBoundingVolume` (→ [BoundingVolume.md](BoundingVolume.md))
**Inherited by:** (none)

A center point and a radius — always a true sphere, never an ellipsoid. The
cheapest-to-test bounding shape in the module, and the default `BoundsType`
most auto-generated node bounds use.

## Behavior notes

- **`extend_by_box()`/`extend_by_hexahedron()`/`extend_by_finite()` all
  reduce to `extend_by_box()`** by first converting the other volume to an
  axis-aligned `BoundingBox` (via its `get_min()`/`get_max()`) and finding
  the minimum radius needed to reach all 8 corners — this is correct but not
  tight for non-box shapes (an oriented hexahedron gets over-approximated by
  its AABB before being folded into the sphere).
- **`around_finite()` (building a sphere around a mixed list of finite
  volumes) special-cases spheres it finds in the list** — it first computes
  a candidate center from the AABB of everything, then if any list member is
  itself a `BoundingSphere`, considers "lop off the corner and use the
  sphere's own radius from center" as a potentially tighter fit than
  treating it as a generic box; non-sphere members still go through the
  box-corner test. If it can't determine a finite volume for some list
  member (`as_finite_bounding_volume()` returns `nullptr`), it gives up and
  calls `set_infinite()` rather than erroring.
- **A single-point `around()` produces a zero-radius, *non-empty* sphere.**
  This mirrors `BoundingVolume::is_empty()`'s note: a zero-radius sphere
  contains exactly the center point, which is a different state from
  `is_empty()` (contains nothing). Building around zero points instead
  yields the actual empty state.
- **NaN points are silently skipped (in debug builds) when building around a
  point list**, with a warning logging how many were skipped — release
  builds (`NDEBUG`) don't perform the check at all.
- **`contains_lineseg()` solves a quadratic for the line/sphere
  intersection** and distinguishes tangent (`radical` nearly zero, single
  root) from the general two-root case; `t1`/`t2` outside `[0, 1]` on both
  sides means the infinite line hits the sphere but the finite segment
  doesn't (`IF_no_intersection`), one endpoint inside means partial overlap
  (`IF_possible|IF_some`), both t-values bracketing `[0, 1]` means the whole
  segment could be inside (`IF_possible|IF_some|IF_all`).
- **`xform()` handles non-uniform scale by taking the *largest* axis
  scale.** It computes the squared length of each of the matrix's three row
  vectors, takes the max, and scales the radius by that — meaning a sphere
  transformed by a non-uniform scale grows enough to still enclose the
  original sphere in every direction, at the cost of no longer being a tight
  fit along the smaller axes. If the resulting radius comes out infinite
  (`cinf()`), the sphere flips to the infinite state rather than holding a
  literal `inf` radius.
- **`contains_box()`/`contains_hexahedron()`/`contains_line()`/
  `contains_plane()` all just forward to the other volume's own
  `contains_sphere(this)` and mask off `IF_all`** — since "is this box
  inside my sphere" is naturally computed as "does my sphere's `contains_box`
  logic, run from the box's perspective, agree," and `IF_all` isn't
  necessarily preserved by swapping which side is queried.
- Allocation goes through `ALLOC_DELETED_CHAIN(BoundingSphere)` (a pooled
  deleted-object freelist macro used throughout Panda for hot allocation
  paths) rather than plain `new`/`delete` bookkeeping.

## API

| Signature | Notes |
|---|---|
| `BoundingSphere()` | Empty sphere |
| `explicit BoundingSphere(const LPoint3 &center, PN_stdfloat radius)` | |
| `LPoint3 get_center() const` / `void set_center(const LPoint3&)` | `MAKE_PROPERTY center` |
| `PN_stdfloat get_radius() const` / `void set_radius(PN_stdfloat)` | `MAKE_PROPERTY radius` |
| `virtual LPoint3 get_min() const` / `get_max() const` | Axis-aligned corners of the sphere's bounding cube |
| `virtual PN_stdfloat get_volume() const` | `4/3 * pi * r^3` |
| `virtual LPoint3 get_approx_center() const` | Exactly `get_center()` for this shape |
| `virtual void xform(const LMatrix4 &mat)` | See behavior notes re: non-uniform scale |
| `virtual void output(std::ostream&) const` | `"bsphere, c (...), r ..."` |

## Usage

```cpp
PT(BoundingSphere) bs = new BoundingSphere(LPoint3(0, 0, 1), 3.0);
if (bs->contains(LPoint3(0, 0, 3)) & BoundingVolume::IF_some) {
  // point is inside (or on) the sphere
}
```

## See also

[BoundingVolume.md](BoundingVolume.md) · [BoundingBox.md](BoundingBox.md) ·
[BoundingHexahedron.md](BoundingHexahedron.md) · [README.md](README.md)
