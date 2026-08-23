# CollisionInvSphere

**Source:** `panda/src/collide/collisionInvSphere.h` (+ `.I`, `.cxx`)
**Inherits:** [CollisionSphere](CollisionSphere.md)

"An inverted sphere: this is a sphere whose collision surface is the inside
surface of the sphere. Everything outside the sphere is solid matter;
everything inside is empty space. Useful for constraining objects to remain
within a spherical perimeter." A world-boundary cage, effectively — pair
with [CollisionHandlerPusher](CollisionHandlerPusher.md) to keep a character
from wandering past a spherical play-area edge.

## Behavior notes

- **Inherits `set_center()`/`set_radius()` from
  [CollisionSphere](CollisionSphere.md) unchanged** — only the intersection
  tests and visualization are flipped inside-out; the constructor arguments
  and accessors mean the same thing as an ordinary sphere.
- **Not itself usable as a *from* solid in most pairings** — like
  [CollisionSphere](CollisionSphere.md), it overrides
  `test_intersection_from_*()` (it's tested *as* the into-shape), but it does
  not override the public `test_intersection()` dispatcher the way
  [CollisionRay](CollisionRay.md)/[CollisionSegment](CollisionSegment.md) do,
  so it's meant as a static boundary (*into*) rather than a moving collider.

## API

| Signature | Notes |
|---|---|
| `explicit CollisionInvSphere(const LPoint3 &center, PN_stdfloat radius)` | |
| `explicit CollisionInvSphere(PN_stdfloat cx, PN_stdfloat cy, PN_stdfloat cz, PN_stdfloat radius)` | |

All other accessors inherited from [CollisionSphere](CollisionSphere.md).

## Usage

```cpp
PT(CollisionInvSphere) boundary = new CollisionInvSphere(0, 0, 0, 100.0);
PT(CollisionNode) cnode = new CollisionNode("world-boundary");
cnode->add_solid(boundary);
render.attach_new_node(cnode);
```

## See also

[CollisionSphere.md](CollisionSphere.md) · [CollisionHandlerPusher.md](CollisionHandlerPusher.md)
· [README.md](README.md)
