# BoundingPlane

**Source:** `panda/src/mathutil/boundingPlane.h` (+ `.I`, `.cxx`)
**Inherits:** `GeometricBoundingVolume` (→ [BoundingVolume.md](BoundingVolume.md))
**Inherited by:** (none)

An infinite half-space, wrapping a single [Plane.md](Plane.md) (`LPlane`).
"Inside" the bounding volume is the side *behind* the plane's normal
(`dist_to_plane() <= 0`); "outside" is in front of the normal. Not a
`FiniteBoundingVolume` — like [BoundingLine](BoundingLine.md), an infinite
half-space has no min/max corner.

## Behavior notes

- **`extend_by_plane()` on a non-empty plane immediately goes infinite** —
  same reasoning as `BoundingLine::extend_by_line()`: two different planes
  don't union into a third plane, so the operation just gives up.
- **`contains_sphere()`/`contains_box()`/`contains_hexahedron()` all wrap
  the other shape in a bounding sphere (box/hexahedron) first and measure
  `dist_to_plane()` from that sphere's center against its radius** — cheap
  reject/accept, then (for box/hexahedron) a per-corner refinement pass
  (`all_in`/`all_out` over the 8 corners) only when the cheap sphere test
  was ambiguous, same two-tier pattern seen in
  [BoundingBox.md](BoundingBox.md)'s `contains_hexahedron()`.
- **`contains_line()` always returns exactly `IF_possible`, unconditionally**
  — a plane-vs-infinite-line test isn't implemented beyond "well, maybe."
- **`contains_plane()` classifies the other plane by the dot product of the
  two normals against `1.0`/`-1.0` (parallel, same direction / parallel,
  opposite direction / not parallel)**: same-direction parallel planes
  compare their `d` term (`get_w()`) to determine which is the more
  restrictive half-space and thus whether one fully contains the other;
  opposite-direction parallel planes either don't intersect at all or
  overlap in a slab; non-parallel planes always intersect somewhere
  (`IF_possible|IF_some`), since two non-parallel infinite planes always
  share a line.
- **`xform()` just delegates to `LPlane::xform()`** — no special handling
  beyond checking `!is_empty() && !is_infinite()` first; see
  [Plane.md](Plane.md) for exactly how a plane transforms (normal
  transforms differently from a point under non-uniform scale/shear —
  `xform_vec_general()`, not a plain vector transform).

## API

| Signature | Notes |
|---|---|
| `BoundingPlane(const LPlane &plane)` | |
| `const LPlane &get_plane() const` | `MAKE_PROPERTY plane` (read-only) |
| `virtual LPoint3 get_approx_center() const` | `_plane.get_point()` — an arbitrary point in the plane |
| `virtual void xform(const LMatrix4 &mat)` | Delegates to `LPlane::xform()` |
| `virtual void output(std::ostream&) const` | `"bplane: ..."` |

## Usage

```cpp
LPlane ground(LVector3(0, 0, 1), LPoint3(0, 0, 0));  // normal up, through origin
PT(BoundingPlane) floor_half_space = new BoundingPlane(ground);
// contains() something below the ground plane -> IF_possible|IF_some
```

## See also

[BoundingVolume.md](BoundingVolume.md) · [Plane.md](Plane.md) ·
[BoundingLine.md](BoundingLine.md) · [README.md](README.md)
