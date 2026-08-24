# CollisionEntry

**Source:** `panda/src/collide/collisionEntry.h` (+ `.I`, `.cxx`)
**Inherits:** `TypedWritableReferenceCount`

"Defines a single collision event. One of these is created for each
collision detected by a [CollisionTraverser](CollisionTraverser.md), to be
dealt with by the [CollisionHandler](CollisionHandler.md)." Provides slots
for a number of values that may or may not be known for any given
collision — it's up to the handler to check `has_*()` before reading.

## Behavior notes

- **Two distinct point/normal pairs, not one.** "Surface" (`surface_point`,
  `surface_normal`) is the point on the *into* solid's boundary itself;
  "contact" (`contact_pos`, `contact_normal`) is where the *from* solid's
  volume actually rests against it (relevant for solids with thickness, like
  spheres/capsules, where the surface intersection point and the resting
  contact point differ by the radius). "Interior point" is a point known to
  be inside the *into* solid, used by handlers that need to compute a push-out
  direction. All three are independently optional — check `has_surface_point()`,
  `has_surface_normal()`, `has_interior_point()`, `has_contact_pos()`,
  `has_contact_normal()` first.
- **Points/normals are stored in the *into* solid's coordinate space and
  converted on read.** `get_surface_point(space)` etc. take a `NodePath`
  specifying which space to return the value in (commonly
  `entry->get_into_node_path()` or `render`) — there's no way to read the
  raw stored value without going through this conversion.
- **`get_t()` is the parametric collision time**, mainly meaningful for
  swept from-solids like [CollisionParabola](CollisionParabola.md); `0` for
  most static shape/shape tests.
- **`get_respect_prev_transform()` mirrors the traverser's setting at the
  time this entry was generated** — reflects whether swept collision was
  active for this hit, not a value set directly on the entry.
- **Clip planes:** `get_into_clip_planes()` lazily discovers and caches
  (`check_clip_planes()`) any `ClipPlaneAttrib` in effect on the *into* node
  path, since [CollisionPolygon](CollisionPolygon.md)/[CollisionBox](CollisionBox.md)
  clip their shape against active clip planes before testing.

## API

### Identity
| Signature | Notes |
|---|---|
| `const CollisionSolid *get_from() const` | The moving/testing solid |
| `bool has_into() const` / `const CollisionSolid *get_into() const` | The struck solid (absent for a Geom-triangle hit, see [CollisionPolygon.md](CollisionPolygon.md)) |
| `CollisionNode *get_from_node() const` / `PandaNode *get_into_node() const` | |
| `NodePath get_from_node_path() const` / `NodePath get_into_node_path() const` | |
| `bool collided() const` / `void reset_collided()` | Distinguishes "confirmed collision" entries from a handler's own bookkeeping-only entries |

### Geometry (space-relative)
| Signature | Notes |
|---|---|
| `LPoint3 get_surface_point(const NodePath &space) const` / `bool has_surface_point() const` / `void set_surface_point(const LPoint3&)` | |
| `LVector3 get_surface_normal(const NodePath &space) const` / `bool has_surface_normal() const` / `void set_surface_normal(const LVector3&)` | |
| `LPoint3 get_interior_point(const NodePath &space) const` / `bool has_interior_point() const` / `void set_interior_point(const LPoint3&)` | |
| `LPoint3 get_contact_pos(const NodePath &space) const` / `bool has_contact_pos() const` / `void set_contact_pos(const LPoint3&)` | |
| `LVector3 get_contact_normal(const NodePath &space) const` / `bool has_contact_normal() const` / `void set_contact_normal(const LVector3&)` | |
| `bool get_all(const NodePath &space, LPoint3 &surface_point, LVector3 &surface_normal, LPoint3 &interior_point) const` | Batch fetch; returns `false` if any piece is missing |
| `bool get_all_contact_info(const NodePath &space, LPoint3 &contact_pos, LVector3 &contact_normal) const` | |
| `PN_stdfloat get_t() const` / `void set_t(PN_stdfloat)` | Parametric collision time |

## Usage

```cpp
for (int i = 0; i < queue->get_num_entries(); ++i) {
  CollisionEntry *entry = queue->get_entry(i);
  if (entry->has_surface_point()) {
    LPoint3 hit = entry->get_surface_point(render);
    // ...
  }
}
```

## See also

[CollisionTraverser.md](CollisionTraverser.md) · [CollisionSolid.md](CollisionSolid.md)
· [CollisionHandler.md](CollisionHandler.md) · [README.md](README.md)
