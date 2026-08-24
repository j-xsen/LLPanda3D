# BoundingBox

**Source:** `panda/src/mathutil/boundingBox.h` (+ `.I`, `.cxx`)
**Inherits:** `FiniteBoundingVolume` (→ [BoundingVolume.md](BoundingVolume.md))
**Inherited by:** (none)

An axis-aligned bounding box (AABB): a min corner and a max corner, always
aligned to the coordinate axes. If you need an oriented/skewed box (e.g. a
view frustum, which is a box in clip space but not axis-aligned once
transformed into world space), use [BoundingHexahedron](BoundingHexahedron.md)
instead.

## Behavior notes

- **`get_point(n)`/`get_num_points()` (8 corners) and `get_plane(n)`/
  `get_num_planes()` (6 faces, as `LPlane`) let a `BoundingBox` be walked
  like a generic convex polyhedron** — used by other bounding volumes
  (`BoundingSphere::extend_by_box()`, `BoundingPlane::contains_box()`) that
  need per-corner or per-face tests without hand-rolling AABB-specific math.
  `get_point()` exploits `_min`/`_max` being consecutive in memory (`&_min`
  indexed with bit tricks) rather than an 8-entry table.
- **`contains_lineseg()` uses a Cohen-Sutherland-style outcode test**: each
  endpoint gets a 6-bit "which face plane(s) is it on the wrong side of"
  code (`a_bits`/`b_bits`). Shared bits between the two codes mean both
  points are outside the same face plane → definitely no intersection
  (cheap reject). Zero bits on both → segment wholly inside → `IF_all`. One
  endpoint's code all-zero → partial overlap. Otherwise it falls back to
  checking whether the *codes* differ across exactly one axis-pair
  (`differ == 0x03/0x0c/0x30`, meaning the segment stretches straight across
  the box along one axis) for a "partial" answer, or gives up with the
  weakest true answer `IF_possible` (correct but not tight — the classic
  Cohen-Sutherland "ambiguous, would need actual clipping to resolve" case).
- **`contains_hexahedron()` tries the cheap AABB-vs-AABB test first
  (`contains_finite()`), and only falls back to the hexahedron's own
  (pricier) box-containment test if the cheap test was inconclusive**
  (neither `IF_no_intersection` nor `IF_all`) — a two-tier "fast path, slow
  path" pattern worth noting if profiling collision/culling code.
- **`contains_line()`/`contains_plane()` both just forward to the other
  side's `contains_box(this)` and strip `IF_all`** — same "ask the other
  shape's own `contains_X` method with the roles swapped" pattern seen in
  [BoundingSphere.md](BoundingSphere.md).
- **`xform()` transforms all 8 corners and re-fits a new AABB around the
  result** — meaning a rotated box's AABB grows (never tight after a
  non-axis-aligned rotation); there is no attempt to detect axis-aligned
  transforms as a special case.
- **NaN handling in `around_points()`** mirrors `BoundingSphere`'s: skipped
  and warned about in debug builds, unchecked in `NDEBUG` builds.
- Allocation goes through `ALLOC_DELETED_CHAIN(BoundingBox)`.

## API

| Signature | Notes |
|---|---|
| `BoundingBox()` | Empty box |
| `explicit BoundingBox(const LPoint3 &min, const LPoint3 &max)` | Debug builds assert `min <= max` componentwise |
| `void set_min_max(const LPoint3 &min, const LPoint3 &max)` | |
| `const LPoint3 &get_minq() const` / `get_maxq() const` | Inline, non-virtual fast accessors (vs. virtual `get_min()`/`get_max()`) |
| `int get_num_points() const` (=8) / `LPoint3 get_point(int n) const` | `MAKE_SEQ points` |
| `int get_num_planes() const` (=6) / `LPlane get_plane(int n) const` | `MAKE_SEQ planes`, each an outward-facing `LPlane` |
| `virtual PN_stdfloat get_volume() const` | `(max-min).x * .y * .z` |
| `virtual LPoint3 get_approx_center() const` | `(min+max)/2`, exact for a box |
| `virtual void xform(const LMatrix4 &mat)` | Re-fits AABB around transformed corners; see notes |
| `virtual void output(std::ostream&) const` | `"bbox, (...) to (...)"` |

## Usage

```cpp
PT(BoundingBox) box = new BoundingBox(LPoint3(-1, -1, -1), LPoint3(1, 1, 1));
for (const LPoint3 &corner : box->get_points()) {
  // walk all 8 corners
}
```

## See also

[BoundingVolume.md](BoundingVolume.md) · [BoundingSphere.md](BoundingSphere.md) ·
[BoundingHexahedron.md](BoundingHexahedron.md) · [Plane.md](Plane.md) (for `LPlane`) ·
[README.md](README.md)
