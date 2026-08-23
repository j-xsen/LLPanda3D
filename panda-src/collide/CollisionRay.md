# CollisionRay

**Source:** `panda/src/collide/collisionRay.h` (+ `.I`, `.cxx`)
**Inherits:** [CollisionSolid](CollisionSolid.md)
**Inherited by:** [CollisionLine](CollisionLine.md)

"An infinite ray, with a specific origin and direction. It begins at its
origin and continues in one direction to infinity, and it has no radius.
Useful for picking from a window, or for gravity effects." The workhorse
*from*-only solid — mouse picking and downward floor/gravity casts are its
two overwhelmingly common uses.

## Behavior notes

- **Meant exclusively as a *from* solid** — it overrides the public
  `test_intersection()` dispatcher itself rather than the
  `test_intersection_from_*()` hooks, meaning other solids test *against* a
  ray, not the reverse. Don't expect a ray to usefully be the *into* target
  of another shape's collider.
- **`set_from_lens(camera, point)` builds the ray directly from a
  `LensNode` (typically the active camera) and a 2D point in the lens's
  `(-1, 1)` screen-space range** — this is the standard way to build a mouse
  picking ray from `mouseWatcherNode`'s current mouse position; no manual
  unprojection math needed.
- **No radius** — [CollisionSegment](CollisionSegment.md) (the finite
  counterpart) is zero-radius too; for a "thick" probe, use a small
  [CollisionCapsule](CollisionCapsule.md)/[CollisionSphere](CollisionSphere.md)
  instead.

## API

| Signature | Notes |
|---|---|
| `CollisionRay()` | Default origin/direction |
| `explicit CollisionRay(const LPoint3 &origin, const LVector3 &direction)` | |
| `explicit CollisionRay(PN_stdfloat ox, PN_stdfloat oy, PN_stdfloat oz, PN_stdfloat dx, PN_stdfloat dy, PN_stdfloat dz)` | |
| `void set_origin(const LPoint3&)` / `const LPoint3 &get_origin() const` | |
| `void set_direction(const LVector3&)` / `const LVector3 &get_direction() const` | |
| `bool set_from_lens(LensNode *camera, const LPoint2 &point)` | Build from screen-space point; `false` if the lens can't produce a ray (e.g. non-invertible projection) |
| `bool set_from_lens(LensNode *camera, PN_stdfloat px, PN_stdfloat py)` | |

## Usage

```cpp
PT(CollisionRay) pick_ray = new CollisionRay();
// each frame, with a valid mouse position:
pick_ray->set_from_lens(camera_node, mouse_watcher->get_mouse());
```

## See also

[CollisionSolid.md](CollisionSolid.md) · [CollisionLine.md](CollisionLine.md)
· [CollisionHandlerQueue.md](CollisionHandlerQueue.md) · [README.md](README.md)
