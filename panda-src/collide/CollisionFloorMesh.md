# CollisionFloorMesh

**Source:** `panda/src/collide/collisionFloorMesh.h` (+ `.I`, `.cxx`)
**Inherits:** [CollisionSolid](CollisionSolid.md)

"An object that represents a solid made entirely of triangles, which will
only be tested again[st] z-axis aligned rays." A cheap terrain/multi-floor
mesh for pure vertical raycasting (walking on uneven ground with many
triangles) without paying for a general triangle-vs-arbitrary-shape test —
narrower than [CollisionPolygon](CollisionPolygon.md) in what it supports,
in exchange for handling many triangles in one solid.

## Behavior notes

- **Only implements `test_intersection_from_ray()` and
  `test_intersection_from_sphere()`** — no line/segment/capsule/box/parabola
  support; it exists specifically for the "cast a ray straight down to find
  floor height" use case (pair with
  [CollisionHandlerFloor](CollisionHandlerFloor.md)/[CollisionHandlerGravity](CollisionHandlerGravity.md)),
  not general-purpose collision.
- **Each triangle caches its own 2D XY bounding box** (`TriangleIndices`:
  `min_x`/`max_x`/`min_y`/`max_y`) alongside its 3 vertex indices, letting
  the ray test cheaply reject most triangles before doing the real
  intersection math.
- **Vertices and triangles are built incrementally**: `add_vertex()` appends
  to a shared vertex pool, `add_triangle(pointA, pointB, pointC)` references
  three vertex indices and computes/caches that triangle's bounding box.

## API

| Signature | Notes |
|---|---|
| `CollisionFloorMesh()` | |
| `void add_vertex(const LPoint3&)` | |
| `void add_triangle(unsigned int pointA, unsigned int pointB, unsigned int pointC)` | Indices into the vertex pool |
| `unsigned int get_num_vertices() const` / `const LPoint3 &get_vertex(unsigned int) const` | |
| `unsigned int get_num_triangles() const` / `LPoint3i get_triangle(unsigned int) const` | |

## Usage

```cpp
PT(CollisionFloorMesh) floor = new CollisionFloorMesh();
floor->add_vertex(LPoint3(0, 0, 0));
floor->add_vertex(LPoint3(10, 0, 1));
floor->add_vertex(LPoint3(0, 10, 0.5));
floor->add_triangle(0, 1, 2);
```

## See also

[CollisionSolid.md](CollisionSolid.md) · [CollisionHandlerFloor.md](CollisionHandlerFloor.md)
· [CollisionHandlerGravity.md](CollisionHandlerGravity.md) · [README.md](README.md)
