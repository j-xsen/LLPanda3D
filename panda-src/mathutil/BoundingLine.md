# BoundingLine

**Source:** `panda/src/mathutil/boundingLine.h` (+ `.I`, `.cxx`)
**Inherits:** `GeometricBoundingVolume` (→ [BoundingVolume.md](BoundingVolume.md))
**Inherited by:** (none)

An infinite line with no thickness, extending in both directions. Not a
`FiniteBoundingVolume` — a line has no min/max corner. The constructor takes
two points, but per the header comment they are *not* endpoints; they're
just two arbitrary points used to define the line's direction, and the line
extends infinitely past both.

## Behavior notes

- **Only a handful of `contains_*`/`extend_by_*` overrides exist**
  (`extend_by_line`, `contains_sphere`, `contains_box`) — everything else
  falls through to `GeometricBoundingVolume`'s generic
  `extend_by_geometric()`/`contains_geometric()` fallbacks, which log a
  warning and force `infinite`/return `IF_dont_understand`. In practice a
  `BoundingLine` is mostly useful as the *into* side of a sphere/box test
  (i.e. "does this line pass near this bounding volume"), not as a general
  combinable bounding volume.
- **`extend_by_line()` on a non-empty line immediately goes infinite** —
  two distinct lines don't generally union into a third line, so
  `extend_by()`'ing a second, different line just gives up and marks the
  volume infinite rather than attempting any kind of "smallest enclosing
  line" computation (which usually wouldn't be a line at all, geometrically).
- **`contains_sphere()`/`contains_box()` both reduce to
  `sqr_dist_to_line()`**, the squared point-to-line distance computed via
  the same "distance from point to infinite line" quadratic-form technique
  as `BoundingSphere::contains_lineseg()`'s discriminant math, but simplified
  since the target here is a point, not a segment intersection. A box is
  tested against its own center and a radius derived from its diagonal
  (`(max-min).length_squared() * 0.25`), so `contains_box()` can only ever
  return `IF_possible` at best (never `IF_all`/`IF_some` — a box can't be
  "inside" an infinitely thin, infinitely long line).
- **`xform()` re-normalizes the direction vector after transforming it, and
  drops to empty if that normalization fails** (i.e. the transform scaled
  the direction to zero length) — the one way a `BoundingLine` can become
  empty after construction.
- The two constructor points are stored internally as `_origin` (point A)
  and `_vector` (`B - A`, implicitly normalized via `xform()`'s
  post-transform renormalization, though the constructor itself doesn't
  explicitly normalize — see `boundingLine.I` if exact endpoint semantics
  matter).

## API

| Signature | Notes |
|---|---|
| `explicit BoundingLine(const LPoint3 &a, const LPoint3 &b)` | `a`/`b` are two points *on* the line, not endpoints |
| `const LPoint3 &get_point_a() const` / `LPoint3 get_point_b() const` | `b` is reconstructed as `origin + vector` |
| `virtual LPoint3 get_approx_center() const` | Midpoint of the two constructor points |
| `virtual void xform(const LMatrix4 &mat)` | Re-normalizes direction; can drop to empty (see notes) |
| `virtual void output(std::ostream&) const` | `"bline, (...) - (...)"` |

## Usage

```cpp
PT(BoundingLine) line = new BoundingLine(LPoint3(0, 0, 0), LPoint3(0, 1, 0));
int r = line->contains(some_sphere);  // "does the sphere touch this infinite line"
```

## See also

[BoundingVolume.md](BoundingVolume.md) · [BoundingPlane.md](BoundingPlane.md) ·
[BoundingSphere.md](BoundingSphere.md) · [README.md](README.md)
