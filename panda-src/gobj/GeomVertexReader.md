# GeomVertexReader

**Source:** `panda/src/gobj/geomVertexReader.h` (+ `.I`, `.cxx`)
**Inherits:** [GeomEnums](README.md#shared-enums-geomenums) **Inherited by:** [GeomVertexRewriter](GeomVertexRewriter.md)

A lightweight, stack-allocated cursor for reading a single named column
(e.g. `"vertex"`, `"normal"`, `"texcoord"`) out of a
[GeomVertexData](GeomVertexData.md) or a standalone
[GeomVertexArrayData](GeomVertexArrayData.md), one row at a time. It's the
standard way application/loader code pulls typed values out of vertex
data without hand-computing byte offsets. It's optimized for reading straight
down one column across many rows — using a separate `GeomVertexReader` per
column and advancing all of them together is faster than repeatedly calling
`set_column()` on one reader to walk across columns at a fixed row.

## Behavior notes

- **Does not hold a reference count on the vertex buffer.** It grabs the
  current data pointer from the `GeomVertexData`/`GeomVertexArrayData` at
  `set_column()` time. Don't keep a `GeomVertexReader` alive across an
  operation that might reallocate or deallocate the underlying buffer, and
  don't hold one long-term — it's meant for one quick pass.
- **Construction order matters when mixing with a writer on the same data:**
  any `GeomVertexWriter`s on the data must be created *before* creating
  readers, since a writer's `set_column()` can trigger a copy-on-write
  reallocation of the buffer, which would invalidate a reader constructed
  first. When both a reader and a writer are needed on the same column,
  [GeomVertexRewriter](GeomVertexRewriter.md) is used instead — it manages
  this ordering internally.
- `set_column()` accepts either a column index or an `InternalName` (via
  `CPT_InternalName`); either form resets `get_read_row()` back to
  `get_start_row()` (the row last passed to `set_row()`, or 0).
- `get_data*()` accessors come in `f`/`d`/plain/`i` families
  (`get_data1f`..`get_data4f`, `get_data1d`..`get_data4d`, `get_data1`..
  `get_data4` which resolve to float or double per `PN_stdfloat`/
  `STDFLOAT_DOUBLE`, and `get_data1i`..`get_data4i`). Each call auto-advances
  to the next row. `get_matrix3f`/`get_matrix4f`(+`d` variants) only work
  when the column's `Contents` is `C_matrix` and it has enough elements —
  they read `N` consecutive same-stride sub-columns as matrix rows.
- **The `force` flag** (default `true`, `set_force()`/`get_force()`)
  controls behavior when the underlying vertex data has been paged out to
  disk (see [VertexDataPage](VertexDataPage.md)): `true` blocks and pages
  the data back in; `false` fails immediately (`set_column()` returns
  `false`) instead, while still queuing the page for later residency. This
  only matters if the application has opted into vertex-data disk paging.
- `is_at_end()` — checking this before each `get_data*()` call is the
  caller's responsibility; reading past the end is undefined behavior (an
  assert in debug builds, garbage/crash in release).
- Debug builds (`_DEBUG`) re-verify on every `quick_set_pointer()`/
  `inc_pointer()` that the stored `_pointer_begin` still matches the
  handle's current read pointer, catching stale-reader-after-reallocation
  bugs early.

## API

| Method | Notes |
|---|---|
| `GeomVertexReader(Thread *thread = current)` | Invalid reader; must be assigned before use. |
| `GeomVertexReader(const GeomVertexData *data, Thread *thread = current)` | Reader with no column set yet — call `set_column()`. |
| `GeomVertexReader(const GeomVertexData *data, CPT_InternalName name, Thread *thread = current)` | Reader pre-bound to a named column. |
| `GeomVertexReader(const GeomVertexArrayData *array, [int column,] Thread *thread = current)` | Reader scoped to one array only, bypassing the full `GeomVertexData`. |
| `get_vertex_data()` / `get_array_data()` | Source object (vertex_data form only / always). |
| `get_array_handle()` | Low-level `GeomVertexArrayDataHandle` for the current array. |
| `get_stride()` | Bytes between consecutive rows. |
| `set_force(bool)` / `get_force()` | See disk-paging note above. |
| `set_column(int column)` / `set_column(CPT_InternalName name)` | Bind to a column by index or name; returns `false` if not found. Resets read row to start row. |
| `clear()` | Reset to the unbound (post-default-construction) state. |
| `has_column()` | Whether a valid column is currently bound. |
| `get_array()` / `get_column()` | Current array index / column descriptor. |
| `set_row(int)` / `set_row_unsafe(int)` | Set read position; `_unsafe` skips the reallocation-safety re-fetch, for performance-critical loops where the array is known unchanged. |
| `get_start_row()` / `get_read_row()` | Row passed to `set_row()` / row the next `get_data*()` will read. |
| `is_at_end()` | True once `get_read_row()` reaches the array's row count. |
| `get_data1f`..`get_data4f`, `get_matrix3f`/`get_matrix4f` | Float-precision reads, auto-advance. |
| `get_data1d`..`get_data4d`, `get_matrix3d`/`get_matrix4d` | Double-precision reads. |
| `get_data1`..`get_data4`, `get_matrix3`/`get_matrix4` | `PN_stdfloat`-precision (build-config dependent). |
| `get_data1i`..`get_data4i` | Integer reads. |
| `output(ostream&)` | Debug string: array, column name/packer, current read row. |

## Usage

```cpp
GeomVertexReader vertex(vdata, InternalName::get_vertex());
GeomVertexReader normal(vdata, InternalName::get_normal());
while (!vertex.is_at_end()) {
  LVecBase3f v = vertex.get_data3f();
  LVecBase3f n = normal.get_data3f();
  // ... process v, n ...
}
```

## See also

- [GeomVertexWriter](GeomVertexWriter.md) — write counterpart
- [GeomVertexRewriter](GeomVertexRewriter.md) — combined reader+writer
- [GeomVertexData](GeomVertexData.md), [GeomVertexArrayData](GeomVertexArrayData.md), [GeomVertexColumn](GeomVertexColumn.md)
