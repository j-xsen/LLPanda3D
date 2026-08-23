# Geom

**Source:** `panda/src/gobj/geom.h` (+ `.I`, `.cxx`)
**Inherits:** CopyOnWriteObject, GeomEnums

A container for geometry primitives. A `Geom` associates one or more
[`GeomPrimitive`](GeomPrimitive.md) objects with a single
[`GeomVertexData`](GeomVertexData.md) table of vertices — each primitive
uses a subset of the same vertex table, and all primitives in a `Geom` are
drawn together in the same graphics state. It's what `GeomNode` (pgraph,
undocumented) holds as its actual renderable payload. `Geom` inherits
`CopyOnWriteObject` (see the module README's "Copy-on-write and interning"
section): it's normally handled through `CPT(Geom)`, and mutating calls
transparently duplicate the object first if it's shared.

## Behavior notes

- **All primitives in a Geom must share one fundamental `PrimitiveType`.**
  You can mix `GeomTriangles` and `GeomTristrips` in the same `Geom`
  (both are `PT_polygons`), but not triangles and points — `get_primitive_type()`
  and `get_shade_model()` return the value common to every primitive added.
- **`decompose()`/`doubleside()`/`reverse()`/`rotate()`/`unify()`/
  `make_points()`/`make_lines()`/`make_patches()`/`make_adjacency()`** are
  all implemented identically: `make_copy()` a new `Geom`, then call the
  corresponding `*_in_place()` mutator on the copy, which in turn calls the
  same-named method on every contained `GeomPrimitive` (see
  [GeomPrimitive.md](GeomPrimitive.md) for what each actually does).
- **Copy-on-write applies to the vertex data too**: `modify_vertex_data()`
  performs COW on the underlying `GeomVertexData` (duplicates only if
  refcounted elsewhere), and separately marks the Geom's own cache/bounds
  stale. `set_vertex_data()` replaces the table outright — validated via
  `check_will_be_valid()` before the assignment (assertion, not exception,
  in release builds this check is skipped).
- **`offset_vertices()`** is specifically for combining multiple Geoms'
  vertex data into one shared big buffer: it replaces the vertex table *and*
  adds a fixed offset to every vertex index referenced by this Geom's
  primitives, so the same physical buffer can serve many Geoms each
  referencing a disjoint (or overlapping) vertex range.
- **Per-Geom munge-result cache (`Geom::Cache`/`Geom::CacheEntry`/
  `Geom::CacheKey`):** every `Geom` keeps its own `pmap` from
  `(source GeomVertexData, GeomMunger)` to a cached munged result, protected
  by its own `_cache_lock` (`LightMutex`). `Geom::CacheEntry` inherits
  [`GeomCacheEntry`](GeomCacheEntry.md) and participates in the global
  [`GeomCacheManager`](GeomCacheManager.md) LRU alongside every other Geom's
  cache entries — that's what actually bounds total cache memory
  (`geom-cache-size`).
- **Pipelined (`CycleData`) storage:** the real per-Geom state
  (`_data`, `_primitives`, `_primitive_type`, `_shade_model`,
  `_geom_rendering`, bounds, `_modified`) lives in a private `Geom::CData`
  cycled through `PipelineCycler<CData>`, following the same
  App/Cull/Draw pipeline-stage pattern used by `RenderState`/`TransformState`
  in pgraph — reads and writes go through `CDReader`/`CDWriter` rather than
  touching fields directly, and there's a *second*, separate cycled
  structure (`Geom::CDataCache`, one per `CacheEntry`) for cached munged
  results specifically, since cache results also need to vary per pipeline
  stage.
- **Bounding volume caching:** `get_bounds()` lazily recomputes
  (`compute_internal_bounds()`) only when `mark_bounds_stale()`/
  `mark_internal_bounds_stale()` has been called since the last computation
  (vertex data changes, primitive changes, `set_bounds_type()` all trigger
  this). `set_bounds()` overrides with an explicit user-supplied volume that
  is never auto-recomputed until `clear_bounds()` is called — note this is
  *opposite* of `PandaNode`'s similar-sounding API (documented as a
  deliberate difference in the header comment for `set_bounds_type()`).
- **`GeomPipelineReader`** is a stack-allocated, non-owning helper (doesn't
  hold a reference to the `Geom` — caller must keep it alive) that
  pre-fetches one pipeline stage's `CData` once and exposes read-only
  accessors + `draw()`/`prepare_now()`, avoiding repeated
  `PipelineCycler` lock/unlock overhead when the caller (typically the
  `CullTraverser`/GSG draw path) needs several fields from the same Geom in
  a tight loop.
- **`~Geom()`** calls both `clear_cache()` and `release_all()` — releasing
  every `GeomContext` this Geom was ever prepared into across every
  `PreparedGraphicsObjects`, mirroring `Texture`'s per-object context
  bookkeeping (see [PreparedGraphicsObjects.md](PreparedGraphicsObjects.md)).

## API

### Construction & identity
| Signature | Notes |
|---|---|
| `explicit Geom(const GeomVertexData *data)` | Only public constructor; a Geom always starts with a vertex table. |
| `Geom *make_copy() const` | Shallow copy — new `Geom`, vertex data/primitives possibly still shared (COW). |
| `PrimitiveType get_primitive_type() const` | Common `PT_*` value across all contained primitives. |
| `ShadeModel get_shade_model() const` | Common `SM_*` value across all contained primitives. |
| `int get_geom_rendering() const` | Bitwise-OR of every primitive's `GeomRendering` requirement bits. |

### Vertex data
| Signature | Notes |
|---|---|
| `CPT(GeomVertexData) get_vertex_data(Thread* = current) const` | Read-only access. |
| `PT(GeomVertexData) modify_vertex_data()` | COW-duplicates if shared; invalidates cache/bounds. |
| `void set_vertex_data(const GeomVertexData *data)` | Replaces table wholesale. |
| `void offset_vertices(const GeomVertexData *data, int offset)` | Replace table + shift every primitive's indices by `offset`. |
| `int make_nonindexed(bool composite_only)` | Converts indexed primitives to nonindexed form where possible. |
| `CPT(GeomVertexData) get_animated_vertex_data(bool force, Thread* = current) const` | Resolves CPU-side vertex animation (see [GeomVertexAnimationSpec](GeomVertexAnimationSpec.md)) if the format needs it. |

### Primitives
| Signature | Notes |
|---|---|
| `size_t get_num_primitives() const` / `get_primitive(i)` / `modify_primitive(i)` | Indexed access to contained `GeomPrimitive`s. |
| `void set_primitive(i, prim)` / `insert_primitive(i, prim)` / `add_primitive(prim)` | Mutators; `add_primitive` = `insert_primitive((size_t)-1, prim)`. |
| `void remove_primitive(i)` / `clear_primitives()` | |
| `bool copy_primitives_from(const Geom *other)` | |

### Transform/decompose operations (each has a `PT(Geom) x()` immutable form and an `x_in_place()` mutator)
| Method | Effect |
|---|---|
| `decompose()` | Break composite primitives (strips/fans) into simple ones. |
| `doubleside()` | Duplicate triangles back-to-back facing opposite directions. |
| `reverse()` | Reverse winding order. |
| `rotate()` | Rotate flat-shaded key vertex to the other end (for shade-model conversion). |
| `unify(max_indices, preserve_order)` | Merge primitives into as few `GeomPrimitive` objects as possible. |
| `make_points()` / `make_lines()` / `make_patches()` / `make_adjacency()` | Convert to the named primitive family. |

### Bounds
| Signature | Notes |
|---|---|
| `CPT(BoundingVolume) get_bounds(Thread* = current) const` | Lazily (re)computed. |
| `int get_nested_vertices(Thread* = current) const` | |
| `void mark_bounds_stale() const` | Force recompute on next `get_bounds()`. |
| `void set_bounds_type(BoundingVolume::BoundsType)` / `get_bounds_type()` | Sphere/box/best/default for the *implicit* volume. |
| `void set_bounds(const BoundingVolume*)` / `clear_bounds()` | Explicit user-supplied volume; not auto-recomputed until cleared. |

### GSG preparation
| Signature | Notes |
|---|---|
| `void prepare(PreparedGraphicsObjects*)` | Queue for preparation. |
| `bool is_prepared(PreparedGraphicsObjects*) const` | |
| `bool release(PreparedGraphicsObjects*)` / `int release_all()` | |
| `GeomContext *prepare_now(PreparedGraphicsObjects*, GraphicsStateGuardianBase*)` | Immediate; normally only called by the GSG. |

### Misc
| Signature | Notes |
|---|---|
| `int get_num_bytes() const` | |
| `UpdateSeq get_modified(Thread* = current) const` / `static UpdateSeq get_next_modified()` | Change-tracking sequence number (primitives/primitive-set only, not vertex data). |
| `bool request_resident() const` | Ask for vertex/index data to be paged back into RAM if paged out. |
| `void transform_vertices(const LMatrix4&)` | |
| `bool check_valid() const` / `check_valid(const GeomVertexData*) const` | |
| `void clear_cache()` / `clear_cache_stage(Thread*)` | Drop this Geom's munge-result cache entries. |
| `calc_tight_bounds(...)` (3 overloads) | Exact min/max point scan, optionally against a named column / with a transform matrix. |

## See also

- [GeomPrimitive](GeomPrimitive.md), [GeomVertexData](GeomVertexData.md),
  [GeomMunger](GeomMunger.md), [GeomCacheEntry](GeomCacheEntry.md),
  [GeomCacheManager](GeomCacheManager.md)
- [PreparedGraphicsObjects](PreparedGraphicsObjects.md),
  [GeomContext](GeomContext.md) — GSG preparation handshake
- [../pgraph/README.md](../pgraph/README.md) — `GeomNode` is what actually
  holds `Geom`s in the scene graph
