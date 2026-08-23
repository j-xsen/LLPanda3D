# GeomVertexData

**Source:** `panda/src/gobj/geomVertexData.h` (+ `.I`, `.cxx`)
**Inherits:** `CopyOnWriteObject`, `GeomEnums` (see [gobj README](README.md#copy-on-write-and-interning) / [shared enums](README.md#shared-enums-geomenums))
**Inherited by:** (none)

The actual numeric vertex data stored in a [Geom](Geom.md), laid out per a
[GeomVertexFormat](GeomVertexFormat.md). Conceptually a single table of
vertices — one row per vertex — physically split across one or more
[GeomVertexArrayData](GeomVertexArrayData.md) arrays (usually just one, with
all columns interleaved). Optionally carries the tables that drive
per-vertex animation: a [TransformTable](TransformTable.md) (hardware
skinning palette), a [TransformBlendTable](TransformBlendTable.md) (CPU
soft-skinning), and a [SliderTable](SliderTable.md) (morph/blend-shape
targets). This is the class you construct directly when building procedural
geometry; you read/write its contents through
[GeomVertexReader/Writer/Rewriter](GeomVertexReader.md), never by poking at
arrays yourself.

## Behavior notes

- **Copy-on-write, not interned** — see
  [gobj README](README.md#copy-on-write-and-interning). Two `GeomVertexData`
  objects with byte-identical content are two separate objects; only
  explicit sharing (assignment, `CPT()` handles) avoids duplication. Most
  mutating calls warn in their doc comments: *"Don't call this in a
  downstream thread unless you don't mind it blowing away other changes you
  might have recently made in an upstream thread"* — a pipelining
  correctness note (see `display`'s pipeline-cycler discussion, not
  reproduced here — the short version: writes from a downstream pipeline
  stage silently clobber upstream, not-yet-flushed changes).
- **`set_format()` reformats existing data**, converting every row into the
  new layout (implemented by literally copying `*this` aside, resetting
  `_format`/`_arrays` to fresh empty arrays for the new format, then calling
  `copy_from()` to re-populate by name-matched column conversion — see
  `convert_to()` below). **`unclean_set_format()`** instead *reinterprets*
  existing bytes under the new format with zero data movement — it asserts
  (debug builds only) that array count and per-array stride match between
  old and new formats, and performs no other validation; get the format
  wrong and you get garbage vertices, silently, in release builds.
- **`convert_to(new_format)`** matches columns name-by-name between formats
  and is itself **cached** per-instance: a private `Cache` keyed by target
  `GeomVertexFormat` (via `CacheKey`/`CacheEntry`, both nested types) stores
  prior conversion results, registered with the global `GeomCacheManager`
  (see [GeomCacheManager](GeomCacheManager.md)) for LRU eviction, so
  repeatedly converting the same source to the same target format (a common
  pattern when GSG-munging the same geometry every frame) is O(1) after
  the first conversion.
- **`copy_from(source, keep_data_objects, ...)`** copies data name-by-name
  between potentially different formats/column types (implicit numeric
  conversion). `keep_data_objects` controls a performance/identity
  tradeoff: `true` retains this object's own `GeomVertexArrayData`
  instances and copies bytes into them in place; `false` instead just
  pointer-copies the source's `GeomVertexArrayData` objects wherever no
  actual conversion is needed (i.e. when the column layouts already match
  byte-for-byte) — faster, but callers holding onto the old array objects
  won't see the update reflected (it's now a different object).
- **`animate_vertices(force, thread)`** is the CPU-skinning entry point,
  called by the GSG/cull code every frame:
  - Short-circuits to `return this` immediately if the format's
    `GeomVertexAnimationSpec::get_animation_type()` isn't `AT_panda` (i.e.
    no animation, or hardware/`AT_hardware` animation, which needs no CPU
    work here).
  - Result is **cached and reused** across calls as long as neither the
    `TransformBlendTable` nor the `SliderTable`'s `get_modified()` sequence
    number has changed — so calling this every frame on unchanged geometry
    is cheap.
  - If `force` is `false` and the vertex data isn't fully resident (see
    `request_resident()`/the disk-paging system in
    [gobj README](README.md#residency-tracking-lrus-and-allocators)), this
    returns the *previous* cached animated result (or the unanimated
    original data if there's no prior cache) rather than blocking — i.e.
    by default you may render one frame of stale/unanimated vertices while
    a paged-out buffer is faulted back in. Pass `force=true` to block until
    resident instead.
- **`pack_abcd`/`unpack_abcd_*`** implement DirectX-style 4-byte-packed
  component packing (`NT_packed_dcba`/`dabc`); **`pack_ufloat`/
  `unpack_ufloat_*`** implement the 3-value 32-bit unsigned-float packing
  used by `NT_packed_ufloat` (11/11/10-bit float triple, same bit layout as
  OpenGL's `R11F_G11F_B10F`) — these are `static` utility methods shared
  with `GeomVertexColumn`'s `Packer` implementations, not something most
  callers use directly.
- `replace_column()` returns a new `GeomVertexData` with one named column's
  type/contents changed (used internally by `scale_color()`/`set_color()`'s
  format-changing overloads) — the new column is appended as a *new array*
  if the old column was interleaved with others, i.e. it does not repack
  the original array in place.

## API

Grouped by purpose.

**Construction / identity**

| Method | Notes |
|---|---|
| `GeomVertexData(name, format, usage_hint)` | `format` must already be registered. |
| `GeomVertexData(const GeomVertexData &copy)` | Shares arrays via COW, no immediate byte copy. |
| `GeomVertexData(const GeomVertexData &copy, format)` | Copy-construct directly into a different format (equivalent to copy + `set_format()`). |
| `int compare_to(other) const` | Content-based ordering (usage hint, then format, then tables, then each array pointer). |
| `const std::string &get_name() const` / `set_name(name)` | Reported on the PStats vertex-computation graph. |

**Format / rows**

| Method | Notes |
|---|---|
| `UsageHint get_usage_hint() const` / `set_usage_hint(hint)` | Propagates to every array. |
| `const GeomVertexFormat *get_format() const` / `set_format(format)` / `unclean_set_format(format)` | See Behavior notes. |
| `bool has_column(name) const` | — |
| `int get_num_rows() const` / `bool set_num_rows(n)` / `unclean_set_num_rows(n)` / `reserve_num_rows(n)` / `clear_rows()` | New rows zero-initialize, except the "color" column which initializes to `(1,1,1,1)`. `unclean_*`/`reserve_*` skip zero-init for a small perf win when you're about to overwrite everything anyway. |
| `size_t get_num_arrays() const` / `CPT(GeomVertexArrayData) get_array(i)` / `PT(...) modify_array(i)` / `set_array(i, array)` | Direct array access — prefer `GeomVertexReader`/`Writer` for row-level work. |

**Animation tables**

| Method | Notes |
|---|---|
| `const TransformTable *get_transform_table() const` / `set_transform_table(t)` / `clear_transform_table()` | Hardware-skinning palette. |
| `CPT(TransformBlendTable) get_transform_blend_table() const` / `modify_transform_blend_table()` / `set_transform_blend_table(t)` / `clear_transform_blend_table()` | CPU soft-skinning (COW-shared table). |
| `const SliderTable *get_slider_table() const` / `set_slider_table(t)` / `clear_slider_table()` | Morph targets; `t` must already be registered. |

**Content operations (each returns a new/converted `GeomVertexData`)**

| Method | Notes |
|---|---|
| `void copy_from(source, keep_data_objects, thread)` | See Behavior notes. |
| `void copy_row_from(dest_row, source, source_row, thread)` | Single-row copy. |
| `CPT(GeomVertexData) convert_to(new_format) const` | Cached; see Behavior notes. |
| `CPT(GeomVertexData) scale_color(scale) const` / `scale_color(scale, num_components, numeric_type, contents) const` | In-place multiply vs. format-changing variant. |
| `CPT(GeomVertexData) set_color(color) const` / `set_color(color, num_components, numeric_type, contents) const` | Same in-place-vs-reformat split. |
| `CPT(GeomVertexData) reverse_normals() const` | Negates the "normal" column; no-op if none present. |
| `CPT(GeomVertexData) animate_vertices(force, thread) const` / `void clear_animated_vertices()` | See Behavior notes. |
| `void transform_vertices(mat)` / `transform_vertices(mat, begin_row, end_row)` / `transform_vertices(mat, const SparseArray &rows)` | Bakes a matrix transform into `C_point`/`C_vector`/`C_normal`/`C_matrix` columns in place. |
| `PT(GeomVertexData) replace_column(name, num_components, numeric_type, contents) const` | See Behavior notes. |

**Misc**

| Method | Notes |
|---|---|
| `int get_num_bytes() const` / `UpdateSeq get_modified(thread) const` | — |
| `bool request_resident() const` | See [VertexDataPage](VertexDataPage.md); non-blocking residency check/fault-in-trigger. |
| `void output(out) const` / `write(out, indent=0) const` / `describe_vertex(out, row) const` | Debug printing; `describe_vertex` dumps every column's value for one row. |
| `void clear_cache()` / `clear_cache_stage()` | Drops the `convert_to()` cache (all pipeline stages, or just the current one). |
| `static uint32_t pack_abcd(a,b,c,d)` / `unpack_abcd_a/b/c/d(data)` / `pack_ufloat(a,b,c)` / `unpack_ufloat_a/b/c(data)` | See Behavior notes. |

`GeomVertexDataPipelineReader`/`GeomVertexDataPipelineWriter` (same header)
are the internal pipeline-stage-scoped accessor classes that most public
`GeomVertexData` methods construct on the fly internally (e.g.
`get_array_info()`, `has_vertex()`/`has_normal()`/`has_color()` and their
`get_*_info()` counterparts, used by the GSG to locate raw column data for
rendering) — application code doesn't normally instantiate these directly.

## Usage

```cpp
CPT(GeomVertexFormat) format = GeomVertexFormat::get_v3n3c4();
PT(GeomVertexData) vdata = new GeomVertexData("triangle", format, Geom::UH_static);
vdata->set_num_rows(3);

GeomVertexWriter vertex(vdata, InternalName::get_vertex());
GeomVertexWriter color(vdata, InternalName::get_color());
vertex.add_data3(0, 0, 0);  color.add_data4(1, 0, 0, 1);
vertex.add_data3(1, 0, 0);  color.add_data4(0, 1, 0, 1);
vertex.add_data3(0, 1, 0);  color.add_data4(0, 0, 1, 1);

PT(GeomTriangles) prim = new GeomTriangles(Geom::UH_static);
prim->add_vertices(0, 1, 2);

PT(Geom) geom = new Geom(vdata);
geom->add_primitive(prim);
```

## See also

- [gobj README](README.md) — COW pattern, shared `GeomEnums`
- [Geom](Geom.md) — pairs this with `GeomPrimitive`s
- [GeomVertexFormat](GeomVertexFormat.md), [GeomVertexArrayData](GeomVertexArrayData.md)
- [GeomVertexReader](GeomVertexReader.md) / GeomVertexWriter / GeomVertexRewriter — actual row-level access
- [TransformTable](TransformTable.md), [TransformBlendTable](TransformBlendTable.md), [SliderTable](SliderTable.md) — animation tables
- [GeomVertexAnimationSpec](GeomVertexAnimationSpec.md) — the format's animation mode, checked by `animate_vertices()`
- [GeomCacheManager](GeomCacheManager.md) — backs the `convert_to()` cache
