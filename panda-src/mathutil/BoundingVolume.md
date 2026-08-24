# BoundingVolume / GeometricBoundingVolume / FiniteBoundingVolume

**Source:** `panda/src/mathutil/boundingVolume.h` (+ `.I`, `.cxx`) ·
`geometricBoundingVolume.h` (+ `.I`, `.cxx`) · `finiteBoundingVolume.h` (+ `.cxx`)
**Inherits:** `BoundingVolume : TypedReferenceCount` ·
`GeometricBoundingVolume : BoundingVolume` ·
`FiniteBoundingVolume : GeometricBoundingVolume`
**Inherited by:** [BoundingSphere](BoundingSphere.md), [BoundingBox](BoundingBox.md),
[BoundingHexahedron](BoundingHexahedron.md) (all `FiniteBoundingVolume`);
[BoundingLine](BoundingLine.md), [BoundingPlane](BoundingPlane.md),
[OmniBoundingVolume](OmniBoundingVolume.md),
[UnionBoundingVolume/IntersectionBoundingVolume](CompositeBoundingVolumes.md)
(all `GeometricBoundingVolume` but not finite)

`BoundingVolume` is the abstract root of every "region of space" type in the
engine — not necessarily geometric (the class comment explicitly allows for
non-spatial bounds), but in practice every concrete subclass in this module
*is* geometric. It's what a scene-graph node's `get_bounds()` returns for
view-frustum culling and what [collide](../collide/README.md)'s
`CollisionSolid::get_bounds()` returns as the cheap pre-filter before a real
shape-vs-shape test — see [CollisionSolid.md](../collide/CollisionSolid.md).

`GeometricBoundingVolume` adds the point-based operations (`extend_by(point)`,
`around(points)`, `contains(point)`, `contains(a, b)` for a line segment) that
only make sense once the volume actually encloses points in 3-D space.
`FiniteBoundingVolume` further narrows that to volumes that are known to be
bounded — `get_min()`/`get_max()` only make sense if the volume can't stretch
to infinity. [BoundingLine](BoundingLine.md) and [BoundingPlane](BoundingPlane.md)
are geometric but *not* finite (a line/plane has no min/max corner), which is
why they derive from `GeometricBoundingVolume` directly rather than
`FiniteBoundingVolume`.

## Behavior notes

- **Every volume carries two independent degenerate states, `empty` and
  `infinite`, tracked as bits in `_flags` (`F_empty`, `F_infinite`) rather
  than as a special value of the volume's own data.** `is_empty()` means
  "contains no points at all" — not the same as a zero-radius sphere, which
  contains exactly one point (its center). `is_infinite()` means "contains
  every point in space." A fresh `BoundingVolume()` starts `F_empty`;
  `set_infinite()` force-sets `F_infinite` and is described as "an infinite
  `extend_by()` operation." Concrete subclasses generally represent both
  states without touching their real data members, so `is_empty()`/
  `is_infinite()` must always be checked before trusting getters like
  `get_center()`/`get_min()` (most assert `!is_empty() && !is_infinite()` in
  debug builds).
- **`extend_by()`/`around()`/`contains()` are all double-dispatch, and the
  dispatch is two levels deep.** `extend_by(vol)` calls `vol->extend_other(this)`
  (virtual on the *argument*, which knows its own concrete type), which in
  turn calls `this->extend_by_sphere(vol)`/`extend_by_box(vol)`/etc. (virtual
  on `this`, now with a concretely-typed argument) — this is the classic
  visitor-pattern double-dispatch trick to get "sphere extended by box" vs.
  "box extended by sphere" behavior without a combinatorial explosion of
  overloads. Each level has a generic fallback
  (`extend_by_finite()`→`extend_by_geometric()`, `around_finite()`→
  `around_geometric()`, `contains_finite()`→`contains_geometric()`) that any
  subclass not implementing a specific pairing falls through to; the
  ultimate `BoundingVolume::extend_by_geometric()`/`around_geometric()`
  fallback logs a `mathutil` warning and forces the receiver `infinite`
  (fail safe, not fail silent) — `contains_geometric()`'s fallback instead
  returns `IF_dont_understand` without mutating anything, since `contains()`
  is a query, not a mutation.
- **`contains()`'s return value is a bitmask of `IntersectionFlags`, not a
  bool, and the bits are structured as increasingly strong claims:**
  `IF_no_intersection` (0, no bits set) means definitely no overlap;
  `IF_possible` means "might intersect" (always set alongside `IF_some`);
  `IF_some` means "definitely intersects"; `IF_all` means the *argument*
  volume is entirely inside `this` one (not symmetric — `IF_all` says
  nothing about whether `this` is inside the argument); `IF_dont_understand`
  means the specific volume-pair combination has no implemented test.
  `BoundingVolume::contains()` short-circuits the empty/infinite cases
  itself before any dispatch: empty-vs-anything is always
  `IF_no_intersection`, `this->is_infinite()` is always
  `IF_possible|IF_some|IF_all`, and `vol->is_infinite()` (with `this` finite)
  is `IF_possible|IF_some` (not `IF_all`, since an infinite volume can't be
  "entirely inside" a finite one).
- **`around(first, last)` resets the volume to exactly enclose a list of
  other volumes, and short-circuits on any infinite member.** It first skips
  leading empty volumes, then scans the rest of the list for `is_infinite()`
  — if any member is infinite, the result is unconditionally infinite
  without ever calling the per-type `around_*` dispatch. Only once a
  concrete non-empty, non-infinite first element is found does it
  double-dispatch through `around_other()` → `around_spheres()`/
  `around_boxes()`/etc., mirroring `extend_by()`'s two-level scheme.
- **`BoundsType` (`BT_default`, `BT_best`, `BT_sphere`, `BT_box`,
  `BT_fastest`) is a config-facing enum, not something a `BoundingVolume`
  instance carries** — it controls *which concrete subclass* something like
  a `GeomNode` should auto-generate for its computed bounds (the
  `bounds-type` prc variable, exposed as `config_mathutil.h`'s
  `bounds_type` `ConfigVariableEnum`). `string_bounds_type()` parses it;
  unrecognized strings quietly fall back to `BT_default` (with an error
  logged by `operator>>` for stream input, but not by
  `string_bounds_type()` itself).
- **The `as_bounding_sphere()`/`as_bounding_box()`/`as_bounding_hexahedron()`/
  `as_bounding_line()`/`as_bounding_plane()`/`as_geometric_bounding_volume()`/
  `as_finite_bounding_volume()` family are the sanctioned RTTI-free downcasts**
  — `BoundingVolume`'s versions all return `nullptr`; each concrete class
  overrides only its own `as_X()` to return `this`. Used pervasively inside
  the `.cxx` files instead of `DCAST`/`dynamic_cast` for the hot
  double-dispatch paths.
- **`GeometricBoundingVolume::around(points)` treats zero points as
  trivially successful** (`_flags = F_empty; if (first != last) { ... }`) —
  distinct from `around()` on an empty *volume list*, which is handled one
  level up in `BoundingVolume::around()`.
- **`GeometricBoundingVolume` disables `PointerToBase`'s type-tracking
  update hook via an explicit template specialization** (`update_type(To*)
  {}` as a no-op) — a low-level memory-accounting detail, not part of the
  public API; mentioned here only because it's easy to trip over if
  grepping the header.
- **`FiniteBoundingVolume::get_volume()` has a non-pure default (unlike
  `get_min()`/`get_max()`, which are pure virtual)** — subclasses that don't
  override it (there is none among the concrete shapes here; all of
  [BoundingSphere](BoundingSphere.md)/[BoundingBox](BoundingBox.md) provide
  their own) would otherwise need one, since [BoundingHexahedron](BoundingHexahedron.md)
  is the one concrete `FiniteBoundingVolume` that does *not* override
  `get_volume()` and so exposes no volume computation at all beyond
  whatever `FiniteBoundingVolume`'s base does (effectively unimplemented for
  hexahedra).

## API

### BoundingVolume — empty/infinite state
| Signature | Notes |
|---|---|
| `bool is_empty() const` / `bool is_infinite() const` | See behavior notes — independent degenerate states |
| `void set_infinite()` | Forces the volume to the infinite state |
| `virtual BoundingVolume *make_copy() const = 0` | Polymorphic copy |

### BoundingVolume — combining/testing
| Signature | Notes |
|---|---|
| `bool extend_by(const BoundingVolume *vol)` | Grows `this` to also enclose `vol` |
| `bool around(const BoundingVolume **first, const BoundingVolume **last)` | Resets `this` to exactly enclose the given list |
| `int contains(const BoundingVolume *vol) const` | Returns an `IntersectionFlags` bitmask |

### BoundingVolume — enums
```cpp
enum IntersectionFlags {
  IF_no_intersection = 0,
  IF_possible        = 0x01,
  IF_some            = 0x02,
  IF_all             = 0x04,
  IF_dont_understand = 0x08
};
enum BoundsType { BT_default, BT_best, BT_sphere, BT_box, BT_fastest };
```
| Signature | Notes |
|---|---|
| `static BoundsType string_bounds_type(const std::string &str)` | Falls back to `BT_default` on unrecognized input |

### BoundingVolume — downcasts (all return `nullptr` except the matching subclass)
| Signature |
|---|
| `virtual GeometricBoundingVolume *as_geometric_bounding_volume()` (+ `const` overload) |
| `virtual const FiniteBoundingVolume *as_finite_bounding_volume() const` |
| `virtual const BoundingSphere *as_bounding_sphere() const` |
| `virtual const BoundingBox *as_bounding_box() const` |
| `virtual const BoundingHexahedron *as_bounding_hexahedron() const` |
| `virtual const BoundingLine *as_bounding_line() const` |
| `virtual const BoundingPlane *as_bounding_plane() const` |

### BoundingVolume — output
| Signature | Notes |
|---|---|
| `virtual void output(std::ostream&) const = 0` / `virtual void write(std::ostream&, int indent_level = 0) const` | `write()` defaults to `indent(...) << *this`; `operator<<` calls `output()` |

### GeometricBoundingVolume — point operations (in addition to the above)
| Signature | Notes |
|---|---|
| `bool extend_by(const LPoint3 &point)` | Overload distinct from `extend_by(BoundingVolume*)` |
| `bool around(const LPoint3 *first, const LPoint3 *last)` | Resets to exactly enclose a point list |
| `int contains(const LPoint3 &point) const` / `int contains(const LPoint3 &a, const LPoint3 &b) const` | Point test / line-segment test, both return `IntersectionFlags` |
| `virtual LPoint3 get_approx_center() const = 0` | Cheap, not-necessarily-exact centroid |
| `virtual void xform(const LMatrix4 &mat) = 0` | Transforms the volume in place |

### FiniteBoundingVolume — bounds
| Signature | Notes |
|---|---|
| `virtual LPoint3 get_min() const = 0` / `virtual LPoint3 get_max() const = 0` | Pure virtual — every finite volume must supply these |
| `virtual PN_stdfloat get_volume() const` | Non-pure default; see behavior notes |

## Usage

```cpp
PT(BoundingSphere) a = new BoundingSphere(LPoint3(0, 0, 0), 2.0);
PT(BoundingBox) b = new BoundingBox(LPoint3(-1, -1, -1), LPoint3(1, 1, 1));

int result = a->contains(b);
if (result & BoundingVolume::IF_some) {
  // a and b overlap (possibly only partially: check IF_all)
}

a->extend_by(b);   // a now also encloses b
```

## See also

[BoundingSphere.md](BoundingSphere.md) · [BoundingBox.md](BoundingBox.md) ·
[BoundingHexahedron.md](BoundingHexahedron.md) · [BoundingLine.md](BoundingLine.md) ·
[BoundingPlane.md](BoundingPlane.md) · [OmniBoundingVolume.md](OmniBoundingVolume.md) ·
[CompositeBoundingVolumes.md](CompositeBoundingVolumes.md) ·
[../collide/CollisionSolid.md](../collide/CollisionSolid.md) (consumer via `get_bounds()`) ·
[README.md](README.md)
