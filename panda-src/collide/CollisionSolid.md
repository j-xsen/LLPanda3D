# CollisionSolid

**Source:** `panda/src/collide/collisionSolid.h` (+ `.I`, `.cxx`)
**Inherits:** `CopyOnWriteObject`
**Inherited by:** [CollisionSphere](CollisionSphere.md), [CollisionBox](CollisionBox.md),
[CollisionCapsule](CollisionCapsule.md), [CollisionPlane](CollisionPlane.md)
(→ [CollisionPolygon](CollisionPolygon.md)), [CollisionRay](CollisionRay.md)
(→ [CollisionLine](CollisionLine.md)), [CollisionSegment](CollisionSegment.md),
[CollisionParabola](CollisionParabola.md), [CollisionFloorMesh](CollisionFloorMesh.md)

Abstract base for "all the things that can collide with other things in the
world, and all the things they can collide with (except geometry)." Never
instantiated directly. A solid holds no scene-graph position by itself — it's
defined in the local coordinate space of whichever [CollisionNode](CollisionNode.md)
holds it, and moves with that node.

## Behavior notes

- **Intersection tests double-dispatch by the *into* solid's C++ type, and
  the pairing is not symmetric.** `test_intersection(entry)` on the *from*
  solid dispatches (via `entry.get_into()->get_type()`, in the `.cxx`) to one
  of the protected `test_intersection_from_sphere/box/line/ray/segment/
  capsule/parabola()` virtuals — each *from*-type overrides only the pairs it
  actually needs to detect. A pair nobody implemented isn't an error: it logs
  once via `report_undefined_intersection_test()`/`report_undefined_from_intersection()`
  (the `collide` notify category) and reports no collision. Practically:
  check each shape's own doc for which "from" role it supports — e.g.
  [CollisionRay](CollisionRay.md) can be a *from* shape, but most solids
  can only be tested *as* the stationary *into* target.
- **`is_tangible()` (default `true`) gates physical response, not
  detection.** An intangible solid still produces `CollisionEntry`s and
  still fires [CollisionHandlerEvent](CollisionHandlerEvent.md) patterns —
  it's invisible only to the `*Physical` handlers' actual movement math.
  Standard way to build a non-blocking "trigger" volume.
- **`effective_normal` overrides the geometric surface normal in results,
  gated by `respect_effective_normal` (default `true`, `respect-effective-normal`
  Config.prc var) on *each individual solid*.** Some shapes (floor-type
  planes/polygons) set this internally to always report straight "up"
  regardless of the facet's actual slope, so a walking character doesn't tip
  over on a slightly-angled floor polygon.
- **Internal bounding volume is lazily computed and cached**, invalidated via
  `mark_internal_bounds_stale()` (any setter that changes shape calls this)
  and rebuilt on next `get_bounds()` by the subclass's
  `compute_internal_bounds()`.
- **Visualization geometry (`_viz_geom`, wireframe/solid/other + a bounds
  variant) is also lazily built** by `fill_viz_geom()`, invalidated via
  `mark_viz_stale()`. This is what [CollisionNode](CollisionNode.md)'s cull
  callback shows when collision visualization is toggled (`show-collision-solids`).
- **`xform(mat)` transforms the solid's own defining points in place** — used
  when a scene graph flattener merges a `CollisionNode` into its parent's
  space; not something game code normally calls directly.

## API

### Tangibility & normal
| Signature | Notes |
|---|---|
| `void set_tangible(bool)` / `bool is_tangible() const` | Physical-response flag; see notes above |
| `void set_effective_normal(const LVector3&)` / `clear_effective_normal()` / `bool has_effective_normal() const` / `const LVector3 &get_effective_normal() const` | Per-solid normal override |
| `void set_respect_effective_normal(bool)` / `bool get_respect_effective_normal() const` | Whether this solid's own effective normal is honored (defaults from the `respect-effective-normal` config var) |

### Bounds & shape
| Signature | Notes |
|---|---|
| `CPT(BoundingVolume) get_bounds() const` / `void set_bounds(const BoundingVolume&)` | Lazily-computed by default; `set_bounds()` overrides with an explicit volume |
| `virtual LPoint3 get_collision_origin() const = 0` | A representative point for the solid (subclass-defined — e.g. sphere center) |
| `virtual CollisionSolid *make_copy() = 0` | Polymorphic copy |
| `virtual void xform(const LMatrix4 &mat)` | Transform the solid's defining points in place |

### Output
| Signature | Notes |
|---|---|
| `virtual void output(std::ostream&) const` / `virtual void write(std::ostream&, int indent_level = 0) const` | Debug printing; `operator<<` calls `output()` |

## Usage

```cpp
PT(CollisionSphere) trigger = new CollisionSphere(LPoint3(0, 0, 0), 2.0);
trigger->set_tangible(false);  // detect-only, don't physically block
```

## See also

[CollisionNode.md](CollisionNode.md) · [CollisionEntry.md](CollisionEntry.md)
· [CollisionTraverser.md](CollisionTraverser.md) · [README.md](README.md)
