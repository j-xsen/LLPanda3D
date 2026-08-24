# Triangulator / Triangulator3

**Source:** `panda/src/mathutil/triangulator.h` (+ `.I`, `.cxx`) ·
`triangulator3.h` (+ `.I`, `.cxx`)
**Inherits:** `Triangulator3 : Triangulator` (no other base for either)
**Inherited by:** (none)

Triangulates an arbitrary simple polygon — convex or concave, and optionally
with holes — into a list of triangles referencing the input vertex indices.
`Triangulator` works purely in 2-D; `Triangulator3` is a thin wrapper that
accepts 3-D points, computes the polygon's best-fit plane, projects
everything into that plane's 2-D coordinate frame, and delegates to the
inherited `Triangulator::triangulate()`.

The core algorithm is a direct port of Narkhede & Manocha's trapezoidation-
based triangulation (UNC-CH, 1994), cited in the header — most of the class's
private members (`seg`, `tr`, `qs`, `mchain`, `vert`, and the large block of
`_max`/`_min`/`_greater_than`/etc. helpers) are internal working state for
that algorithm and are not meant to be used or understood from outside; they
exist here only because they're `protected` rather than hidden in a `.cxx`-
local struct.

## Behavior notes

- **The public workflow is strictly sequential and stateful**: `add_vertex()`
  populates a shared vertex pool (returns an index); `add_polygon_vertex(index)`
  builds the outer boundary as a list of indices into that pool (either
  winding direction is accepted — see `is_left_winding()`); `begin_hole()` +
  repeated `add_hole_vertex(index)` add zero or more interior holes, each
  its own index list; `triangulate()` does the actual work; then
  `get_num_triangles()`/`get_triangle_v0/1/2(n)` read back results as index
  triples into the *same* vertex pool. Calling `triangulate()` again without
  `clear_polygon()`/`clear()` reprocesses the same polygon+hole definition.
- **`clear()` wipes the vertex pool and the polygon/hole definition together;
  `clear_polygon()` wipes only the polygon/hole definition, keeping vertices**
  — useful for triangulating multiple polygons that share a vertex pool.
- **`cleanup_polygon_indices()` silently drops any out-of-range index** from
  the polygon/hole lists before triangulating, rather than asserting — a
  defensive pass against bad input indices.
- **Degenerate input (fewer than 3 polygon vertices after cleanup) produces
  zero triangles, not an error** — `triangulate()` just returns early with
  an empty result.
- **The trapezoidation step can fail and retries with a re-shuffled segment
  order** (`construct_trapezoids()` returning nonzero) — the algorithm's
  numerical robustness depends on the order segments are processed in, so on
  failure a fresh `Randomizer` reshuffles the `permute` index array and the
  whole trapezoidation restarts. A comment in the source notes the
  *initial* shuffle (before any failure) is currently disabled/commented
  out — "not sure why we should shuffle the index... isn't the initial
  order as good as any other?" — so in practice the first attempt normally
  uses index order as-is, and only a genuine numerical failure triggers
  randomization.
- **Triangulator3's plane-fitting uses a "sum of cross products of
  consecutive edges projected onto each major axis" technique to derive the
  polygon normal** (Newell's method), which tolerates mildly non-planar
  input and correctly weights the normal by each projection's implicit area
  — not a simple 3-point cross product. If the polygon is degenerate
  (zero area in every projection — e.g. all points collinear),
  `normal.normalize()` fails and `triangulate()` returns with no triangles,
  same as the base class's "fewer than 3 vertices" case.
- **The 2-D projection basis comes from `heads_up()`** (see
  [LookAt.md](LookAt.md)) using the polygon's own edge `_vertices3[1] -
  _vertices3[2]` as the "forward" direction and the computed normal as
  "up," then inverting that matrix and using it to project all 3-D vertices
  into the plane's local XY — `Triangulator3::get_plane()` exposes the
  resulting `LPlaned` afterward for the caller to reconstruct 3-D positions
  from the 2-D triangulation result if needed.
- **`Triangulator3` re-declares its own `_vertices3`/`get_vertex()`/
  `add_vertex()` rather than reusing the base class's 2-D `_vertices`** —
  the base's `_vertices` is populated internally by `Triangulator3::triangulate()`
  as a *projection* of `_vertices3`, so the two vertex pools are parallel
  but not identical; calling the base class's `add_vertex(LPoint2d)`
  directly on a `Triangulator3` would desync them (not guarded against —
  a caller must be consistent about which `add_vertex()` overload they use).
- All internal geometry uses `double` (`LPoint2d`/`LPoint3d`/`LPlaned`)
  regardless of the engine's `PN_stdfloat` build setting — triangulation is
  precision-sensitive enough that this isn't parameterized like most other
  mathutil types.

## API

### Triangulator (2-D)
| Signature | Notes |
|---|---|
| `void clear()` | Clears vertex pool + polygon/hole definition |
| `int add_vertex(const LPoint2d &point)` / `add_vertex(double x, double y)` | Returns pool index |
| `int get_num_vertices() const` / `const LPoint2d &get_vertex(int n) const` | `MAKE_SEQ vertices` |
| `void clear_polygon()` | Clears only the polygon/hole definition |
| `void add_polygon_vertex(int index)` | Either winding direction accepted |
| `bool is_left_winding() const` | Winding of the polygon as added |
| `void begin_hole()` / `void add_hole_vertex(int index)` | Starts a new hole; must call `begin_hole()` first |
| `void triangulate()` | Does the work; may internally retry on numerical failure |
| `int get_num_triangles() const` | |
| `int get_triangle_v0(int n) const` / `get_triangle_v1(int n) const` / `get_triangle_v2(int n) const` | Vertex-pool indices |

### Triangulator3 (3-D, projects to a plane)
| Signature | Notes |
|---|---|
| `void clear()` | Also resets the cached plane |
| `int add_vertex(const LPoint3d &point)` / `add_vertex(x, y, z)` | 3-D vertex pool (parallel to base's 2-D pool — see notes) |
| `int get_num_vertices() const` / `const LPoint3d &get_vertex(int n) const` | `MAKE_SEQ vertices` |
| `void triangulate()` | Fits a plane, projects, delegates to `Triangulator::triangulate()` |
| `const LPlaned &get_plane() const` | The fitted plane, valid after `triangulate()` |

## Usage

```cpp
Triangulator t;
int a = t.add_vertex(0, 0);
int b = t.add_vertex(4, 0);
int c = t.add_vertex(4, 4);
int d = t.add_vertex(0, 4);
t.add_polygon_vertex(a);
t.add_polygon_vertex(b);
t.add_polygon_vertex(c);
t.add_polygon_vertex(d);
t.triangulate();
for (int i = 0; i < t.get_num_triangles(); ++i) {
  // t.get_triangle_v0(i), v1(i), v2(i) index into the vertex pool
}
```

## See also

[LookAt.md](LookAt.md) (for `heads_up()`, used by `Triangulator3`'s plane fit) ·
[Plane.md](Plane.md) · [README.md](README.md)
