# BoundingHexahedron

**Source:** `panda/src/mathutil/boundingHexahedron.h` (+ `.I`, `.cxx`)
**Inherits:** `FiniteBoundingVolume` (→ [BoundingVolume.md](BoundingVolume.md))
**Inherited by:** (none)

A convex hexahedron: 8 points, 6 planes. Its primary purpose is representing
a camera's view frustum (the `LFrustum`-taking constructor converts a
[Frustum.md](Frustum.md) into 8 corner points), but it's equally usable for
any oriented convex box — an OBB, in contrast to the always-axis-aligned
[BoundingBox](BoundingBox.md).

## Behavior notes

- **Every mutation (`xform()`, both constructors) recomputes the centroid
  and 6 face planes from the 8 corner points** via private `set_centroid()`/
  `set_planes()` — there is no incremental update path; even a single-axis
  `xform()` re-derives all 6 planes from scratch.
- **`BoundingHexahedron(const LFrustum&, is_ortho, cs)` builds the corners in
  Z-up-right space first, unconditionally, then converts coordinate systems
  afterward if needed** (`xform(LMatrix4::convert_mat(CS_zup_right, cs))`
  when `cs != CS_zup_right`) rather than deriving each coordinate system's
  corner layout directly — simpler code, one extra matrix multiply for
  non-default coordinate systems. For a perspective frustum (`is_ortho ==
  false`), the far-plane corners are scaled by `far/near` relative to the
  near-plane corners (`fs = frustum._ffar / frustum._fnear`); for an
  orthographic frustum, `fs = 1.0` (near and far cross-sections are the same
  size).
- **`contains_point()`/`contains_lineseg()`/`contains_sphere()` all test
  against the 6 face planes directly, not the corner points** — a point is
  inside iff `dist_to_plane(point) <= 0` for all six ([Plane.md](Plane.md)'s
  sign convention: negative = behind the plane, i.e. inside the frustum).
  This makes hexahedron-vs-point/sphere tests cheap (6 dot products) but
  means `contains_lineseg()` for a segment that crosses in and out through a
  face without either endpoint being fully rejected returns the weak
  `IF_possible` — the comment admits it "won't bother to check that more
  thoroughly."
- **`contains_box()` puts the box inside a bounding sphere first** (same
  "wrap the other shape in a cheaper primitive" pattern as
  `BoundingPlane::contains_box()`) before doing the per-plane distance test,
  rather than testing all 8 box corners against all 6 planes directly.
- **`FiniteBoundingVolume::get_volume()` is *not* overridden** — a
  `BoundingHexahedron` has no volume computation; calling the inherited
  default (if any) or relying on it is not meaningful for this shape. Use
  `get_min()`/`get_max()`'s AABB volume as a (loose) proxy if needed.
- **`write()` (unlike most bounding volumes) prints every corner point and
  the centroid**, not just min/max — useful for debugging frustum
  construction since a hexahedron's shape isn't fully described by its AABB.

## API

| Signature | Notes |
|---|---|
| `BoundingHexahedron(const LFrustum &frustum, bool is_ortho, CoordinateSystem cs = CS_default)` | Builds from a camera frustum |
| `BoundingHexahedron(const LPoint3 &fll, &flr, &fur, &ful, &nll, &nlr, &nur, &nul)` | Builds from explicit far/near-corner points (far/near, lower/upper, left/right) |
| `int get_num_points() const` (=8) / `LPoint3 get_point(int n) const` | `MAKE_SEQ points` |
| `int get_num_planes() const` (=6) / `LPlane get_plane(int n) const` | `MAKE_SEQ planes` |
| `virtual LPoint3 get_min() const` / `get_max() const` | AABB of the 8 corners |
| `virtual LPoint3 get_approx_center() const` | The cached centroid |
| `virtual void xform(const LMatrix4 &mat)` | Transforms all 8 points, recomputes centroid + planes |
| `virtual void output(std::ostream&) const` / `write(std::ostream&, int) const` | `write()` dumps every corner + centroid |

## Usage

```cpp
LFrustum frustum;
frustum.make_perspective_hfov(60.0, 1.333, 1.0, 1000.0);
PT(BoundingHexahedron) view_frustum =
  new BoundingHexahedron(frustum, false, CS_zup_right);

if (view_frustum->contains(some_bounds) & BoundingVolume::IF_no_intersection) {
  // safe to cull
}
```

## See also

[BoundingVolume.md](BoundingVolume.md) · [Frustum.md](Frustum.md) ·
[BoundingBox.md](BoundingBox.md) · [Plane.md](Plane.md) · [README.md](README.md)
