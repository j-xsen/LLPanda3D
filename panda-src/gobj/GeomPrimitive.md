# GeomPrimitive

**Source:** `panda/src/gobj/geomPrimitive.h` (+ `.I`, `.cxx`)
**Inherits:** CopyOnWriteObject, GeomEnums
**Inherited by:** `GeomPoints`, `GeomLines`, `GeomLinesAdjacency`,
`GeomLinestrips`, `GeomLinestripsAdjacency`, `GeomTriangles`,
`GeomTrianglesAdjacency`, `GeomTristrips`, `GeomTristripsAdjacency`,
`GeomTrifans`, `GeomPatches` — all covered in this file (see
"Concrete subclasses" below; the module README documents the fold-in
decision).

Abstract base for the family of classes representing indexed-topology
geometry primitives stored in a [`Geom`](Geom.md). A `GeomPrimitive` holds
an ordered list of integer indices into a `GeomVertexData`'s vertex table
(or, in the "nonindexed" case, an implicit consecutive range) — what those
indices *mean* (a series of independent triangles vs. a connected strip,
etc.) is defined entirely by the concrete subclass. Like `Geom`, it
inherits `CopyOnWriteObject` (see the module README's "Copy-on-write and
interning" section).

## Behavior notes

- **Nonindexed fast path:** a `GeomPrimitive` starts out with no index
  array (`_vertices` null) and tracks only `_first_vertex`/`_num_vertices`.
  `add_vertex()` keeps it in this compact nonindexed form as long as each
  added vertex is exactly the next consecutive integer; the moment a
  non-consecutive vertex is added, it silently converts to a real indexed
  array (`do_make_indexed()`). `is_indexed()`/`is_composite()` (composite =
  `get_num_vertices_per_primitive() == 0`, i.e. strip/fan-style variable
  length) let calling code branch without knowing the concrete subclass.
- **Index type auto-elevation:** the index array's `NumericType` starts at
  whatever `set_index_type()` specifies (normally `NT_uint16`) and
  `add_vertex()` silently promotes it via `consider_elevate_index_type()`
  — `NT_uint8`→`NT_uint16` at vertex ≥ 0xff, `NT_uint16`→`NT_uint32` at
  vertex ≥ 0xffff. Promoting re-encodes the entire existing index array.
- **Strip-cut / primitive-restart index reservation:** the *highest*
  representable value of each index type (`0xff` for uint8, `0xffff` for
  uint16, `-1`/`0x7fffffff`-ish for uint32) is permanently reserved as the
  "strip cut" (primitive restart) marker and is never a valid vertex index
  — `get_highest_index_value()` returns the type's max minus one for this
  reason, and re-typing an index array remaps any existing strip-cut
  markers to the new type's reserved value.
- **`decompose()`/`rotate()`/`doubleside()`/`reverse()`** dispatch to
  protected virtuals (`decompose_impl()`/`rotate_impl()`/etc.) that each
  concrete subclass overrides only where the operation is meaningful;
  unsupported operations fall back to base-class no-op behavior (e.g.
  `rotate()` returns the *same* primitive unchanged, not a copy, if
  `rotate_impl()` returns null — check for this if you need to know
  whether rotation actually happened).
- **`match_shade_model()`**: `SM_uniform` is compatible with anything;
  `SM_flat_first_vertex` and `SM_flat_last_vertex` are each satisfiable by
  rotating into the other; any other mismatch returns null (incompatible).
- **Composite primitives (variable vertices-per-primitive)** — those
  returning `get_num_vertices_per_primitive() == 0` — compute derived
  counts from three virtuals: `get_min_num_vertices_per_primitive()`
  (smallest legal primitive, e.g. 3 for a tristrip), and
  `get_num_unused_vertices_per_primitive()` (degenerate "unused" connector
  vertices inserted between primitives when `requires_unused_vertices()`
  is true — see per-subclass table below). `get_num_primitives()` for a
  composite primitive equals the length of its `_ends` array (one end
  offset per sub-primitive), not a simple division.
- **`Contexts` map**: like `Geom`, each `GeomPrimitive` tracks every
  `PreparedGraphicsObjects`/`IndexBufferContext` pair it's been prepared
  into, released automatically on destruction.

## API

### Construction, identity, format
| Signature | Notes |
|---|---|
| `explicit GeomPrimitive(UsageHint usage_hint)` | |
| `virtual PT(GeomPrimitive) make_copy() const = 0` | |
| `virtual PrimitiveType get_primitive_type() const = 0` | |
| `virtual int get_geom_rendering() const` | `GeomRendering` bits this primitive requires. |
| `ShadeModel get_shade_model() const` / `set_shade_model(...)` | |
| `UsageHint get_usage_hint() const` / `set_usage_hint(...)` | |
| `NumericType get_index_type() const` / `set_index_type(...)` | See index-elevation note above. |

### Vertex/index iteration (safe, subclass-agnostic)
| Signature | Notes |
|---|---|
| `bool is_composite() const` / `is_indexed() const` | |
| `int get_first_vertex() const` / `get_num_vertices() const` / `get_vertex(i) const` | |
| `void add_vertex(int)` / `add_vertices(v1,v2[,v3[,v4]])` | |
| `void add_consecutive_vertices(start, num)` / `add_next_vertices(num)` | |
| `void reserve_num_vertices(num)` | |
| `bool close_primitive()` | Ends the current sub-primitive (composite types). |
| `void clear_vertices()` | |
| `void offset_vertices(offset[, begin_row, end_row])` | |
| `void make_indexed()` | Force out of the nonindexed fast path. |
| `void make_nonindexed(dest, source)` / `pack_vertices(dest, source)` | Rewrite referenced vertex subset into a fresh compact table. |

### Sub-primitive introspection
| Signature | Notes |
|---|---|
| `int get_num_primitives() const` | Sub-primitive count (see composite note above). |
| `int get_primitive_start(n)` / `get_primitive_end(n)` / `get_primitive_num_vertices(n)` | |
| `int get_num_used_vertices() const` | |
| `int get_num_faces() const` / `get_primitive_num_faces(n)` | |
| `int get_min_vertex() const` / `get_max_vertex() const` (+ per-primitive variants) | |

### Transform/decompose (each returns a new primitive; base no-ops fall back to `this`)
| Method |
|---|
| `decompose()`, `rotate()`, `doubleside()`, `reverse()`, `match_shade_model(sm)`, `make_points()`, `make_lines()`, `make_patches()`, `virtual make_adjacency()` |

### Low-level array access ("not intended for high-level usage" per the header)
| Signature | Notes |
|---|---|
| `CPT(GeomVertexArrayData) get_vertices() const` / `modify_vertices(num = -1)` / `set_vertices(...)` | The raw index array. |
| `int get_index_stride() const` / `get_strip_cut_index() const` | |
| `CPTA_int get_ends() const` / `modify_ends()` / `set_ends(...)` | Per-sub-primitive end offsets (composite types). |
| `set_minmax(...)` / `clear_minmax()` / `get_mins()` / `get_maxs()` | Cached min/max vertex-index bookkeeping. |
| `virtual int get_num_vertices_per_primitive() const` | 0 = composite/variable. |
| `virtual int get_min_num_vertices_per_primitive() const` | |
| `virtual int get_num_unused_vertices_per_primitive() const` | Degenerate connector vertices between sub-primitives. |

### GSG preparation
| Signature | Notes |
|---|---|
| `void prepare(PreparedGraphicsObjects*)` / `is_prepared(...)` | |
| `IndexBufferContext *prepare_now(PreparedGraphicsObjects*, GraphicsStateGuardianBase*)` | |
| `bool release(...)` / `int release_all()` | |
| `static const GeomVertexArrayFormat *get_index_format(NumericType)` | The (interned) array format used for index buffers of a given index type. |

## Concrete subclasses

All eleven are thin: they define `get_primitive_type()`, the
per-primitive vertex-count virtuals, `draw()` (dispatches to the GSG), and
override `decompose_impl()`/`rotate_impl()`/`doubleside_impl()`/
`reverse_impl()`/`requires_unused_vertices()`/`append_unused_vertices()`
only where that operation applies to their topology.

| Class | Topology | Vertices/primitive | Adjacency | Notable overrides |
|---|---|---|---|---|
| `GeomPoints` | Disconnected points | 1 (fixed) | no | — |
| `GeomLines` | Disconnected line segments | 2 (fixed) | no | `rotate_impl()`, `make_adjacency()` |
| `GeomLinesAdjacency` | Disconnected line segments + adjacency | 4 (fixed) | yes | — |
| `GeomLinestrips` | Connected line strips | 0 (composite, min 2) | no | `decompose_impl()`, `rotate_impl()`, degenerate-vertex insertion (1 unused/primitive), `make_adjacency()` |
| `GeomLinestripsAdjacency` | Connected line strips + adjacency | 0 (composite, min 4) | yes | `decompose_impl()`, degenerate-vertex insertion (1 unused/primitive) |
| `GeomTriangles` | Disconnected triangles | 3 (fixed) | no | `doubleside_impl()`, `reverse_impl()`, `rotate_impl()`, `make_adjacency()` |
| `GeomTrianglesAdjacency` | Disconnected triangles + adjacency | 6 (fixed) | yes | `doubleside_impl()`, `reverse_impl()` |
| `GeomTristrips` | Triangle strips | 0 (composite, min 3) | no | `decompose_impl()`, `doubleside_impl()`, `reverse_impl()`, `rotate_impl()`, degenerate-vertex insertion (2 unused/primitive), `make_adjacency()` |
| `GeomTristripsAdjacency` | Triangle strips + adjacency | 0 (composite, min 6) | yes | degenerate-vertex insertion (1 unused/primitive) |
| `GeomTrifans` | Triangle fans | 0 (composite, base defaults) | no | `decompose_impl()`, `rotate_impl()` |
| `GeomPatches` | Fixed-size vertex groupings for tessellation shaders | `_num_vertices_per_patch` (ctor arg, arbitrary fixed N) | no | custom bam I/O (stores patch size) |

Per-class notes:

- **`GeomPoints`/`GeomLines`/`GeomTriangles`/adjacency variants** are the
  "simple" (non-composite) primitives — every sub-primitive uses exactly
  the same fixed vertex count, so `get_num_primitives()` is a plain
  division and no `_ends` array or degenerate-vertex logic is needed.
  `*Adjacency` variants store extra adjacent-geometry vertices interleaved
  with the real ones (per-primitive count doubles: 2→4 for lines, 3→6 for
  triangles) for use by geometry shaders that need to see neighboring
  topology (e.g. silhouette detection).
- **`GeomLinestrips`/`GeomTristrips`/`GeomTrifans`** are composite: each
  sub-primitive's vertex count is read from the `_ends` array.
  `GeomLinestrips`/`GeomTristrips` additionally insert degenerate
  "unused" vertices between consecutive strips when packed into a single
  nonindexed run (`requires_unused_vertices()` true), so strip boundaries
  don't produce a spurious connecting edge; `append_unused_vertices()`
  defines exactly what gets inserted. `GeomTrifans` does not require
  unused vertices (fans can't be chained the same way) and instead relies
  purely on the `_ends` array + `decompose_impl()`.
  `GeomLinestripsAdjacency`/`GeomTristripsAdjacency` extend the same
  pattern with the extra adjacency vertices baked into each strip.
- **`GeomPatches`** is the outlier: it doesn't represent a fixed rendering
  topology at all — `num_vertices_per_patch` is caller-specified per
  instance (e.g. 3 for triangle-equivalent patches, 4 for quad patches,
  16 for bicubic patches), consumed entirely by a tessellation control/
  evaluation shader rather than interpreted by fixed-function hardware.
  Corresponds to `GeomEnums::PT_patches`.
- Only `GeomLines`, `GeomLinestrips`, `GeomTriangles`, and `GeomTristrips`
  implement `make_adjacency()` (converting to their `*Adjacency`
  counterpart) — the `*Adjacency` classes themselves and `GeomPoints`/
  `GeomTrifans`/`GeomPatches` inherit the base no-op.

## See also

- [Geom](Geom.md) — the container that holds a set of `GeomPrimitive`s
  against one `GeomVertexData`
- [GeomVertexData](GeomVertexData.md), [GeomVertexArrayData](GeomVertexArrayData.md),
  [GeomVertexArrayFormat](GeomVertexArrayFormat.md) — vertex/index storage
- [GeomMunger](GeomMunger.md) — converts primitives/vertex data to a
  GSG-friendly layout
- [IndexBufferContext](IndexBufferContext.md),
  [PreparedGraphicsObjects](PreparedGraphicsObjects.md) — GPU preparation
