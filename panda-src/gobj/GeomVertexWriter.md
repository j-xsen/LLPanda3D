# GeomVertexWriter

**Source:** `panda/src/gobj/geomVertexWriter.h` (+ `.I`, `.cxx`)
**Inherits:** [GeomEnums](README.md) **Inherited by:** [GeomVertexRewriter](GeomVertexRewriter.md)

The write counterpart to [GeomVertexReader](GeomVertexReader.md): a
lightweight cursor for writing typed values into a single named column of a
[GeomVertexData](GeomVertexData.md) or standalone
[GeomVertexArrayData](GeomVertexArrayData.md), one row at a time. It's the
standard way to populate procedurally-generated geometry.

## Behavior notes

- **`set_data*()` vs. `add_data*()`:** `set_data*()` overwrites an existing
  row and asserts if it would run past the end of current data.
  `add_data*()` does the same when writing into the middle of existing rows,
  but silently *extends* the array (adding a new row) when it runs past the
  end — this is the family to use when building up geometry from scratch,
  e.g. in a loop calling `add_data3f()` once per new vertex. Mixing the two
  families on one writer works but the distinction only matters at the
  write position currently past the last row.
- Same buffer-lifetime caveat as `GeomVertexReader`: doesn't hold a
  reference count on the vertex buffer, isn't meant to be kept long-term,
  and — critically — **writers must be constructed before readers** on the
  same `GeomVertexData` when both are needed, since `set_column()` on a
  writer can trigger a copy-on-write reallocation that would invalidate an
  already-constructed reader's pointers.
  [GeomVertexRewriter](GeomVertexRewriter.md) is preferred when both are
  needed on the same column.
- `reserve_num_rows(int)` pre-allocates array storage for up to that many
  rows without triggering a reallocation on each `add_data*()` call — worth
  calling before a large procedural-generation loop. When the writer was
  constructed from a full `GeomVertexData` (not a single array), this
  resizes *all* arrays in the format together (rows must stay aligned
  across arrays), going through `GeomVertexDataPipelineWriter::
  reserve_num_rows()`; when constructed from a single
  `GeomVertexArrayData`, only that one array is resized.
- No `force`/disk-paging flag exists here (unlike `GeomVertexReader`) —
  writing always requires the array resident; `set_vertex_column()`
  goes through `GeomVertexDataPipelineWriter`, which pages data in as
  needed.
- Has no `is_at_end()`-driven safety net for `set_data*()` — it's the
  caller's responsibility to stay within `get_write_row() < num_rows`
  when using `set_data*()`, or to use `add_data*()` if the row count is
  meant to grow.

## API

| Method | Notes |
|---|---|
| `GeomVertexWriter(Thread *thread = current)` | Invalid writer; must be assigned before use. |
| `GeomVertexWriter(GeomVertexData *data, Thread *thread = current)` | Writer with no column set — call `set_column()`. |
| `GeomVertexWriter(GeomVertexData *data, CPT_InternalName name, Thread *thread = current)` | Writer pre-bound to a named column. |
| `GeomVertexWriter(GeomVertexArrayData *array, [int column,] Thread *thread = current)` | Writer scoped to one array only. |
| `get_vertex_data()` / `get_array_data()` | Source object. |
| `get_array_handle()` | Low-level `GeomVertexArrayDataHandle`. |
| `get_stride()` | Bytes between consecutive rows. |
| `set_column(int column)` / `set_column(CPT_InternalName name)` | Bind to a column; resets write row to start row. |
| `reserve_num_rows(int num_rows)` | Pre-allocate storage; see note above. |
| `clear()` | Reset to unbound state. |
| `has_column()` | Whether a column is bound. |
| `get_array()` / `get_column()` | Current array index / column descriptor. |
| `set_row(int)` / `set_row_unsafe(int)` | Set write position. |
| `get_start_row()` / `get_write_row()` | Row passed to `set_row()` / row the next `set_data*`/`add_data*` will write. |
| `is_at_end()` | True at the current end of existing data. |
| `set_data1f`..`set_data4f`, `set_matrix3f`/`set_matrix4f` (+`d`, plain, `i` variants) | Overwrite current row, advance; error if past end. |
| `add_data1f`..`add_data4f`, `add_matrix3f`/`add_matrix4f` (+`d`, plain, `i` variants) | Overwrite or extend, advance. |
| `output(ostream&)` | Debug string: array, column name/packer, current write row. |

## Usage

```cpp
PT(GeomVertexData) vdata = new GeomVertexData(
    "triangle", GeomVertexFormat::get_v3n3c4(), Geom::UH_static);
GeomVertexWriter vertex(vdata, InternalName::get_vertex());
GeomVertexWriter color(vdata, InternalName::get_color());
vertex.add_data3f(0, 0, 0); color.add_data4f(1, 0, 0, 1);
vertex.add_data3f(1, 0, 0); color.add_data4f(0, 1, 0, 1);
vertex.add_data3f(0, 1, 0); color.add_data4f(0, 0, 1, 1);
```

## See also

- [GeomVertexReader](GeomVertexReader.md) — read counterpart
- [GeomVertexRewriter](GeomVertexRewriter.md) — combined reader+writer
- [GeomVertexData](GeomVertexData.md), [GeomVertexArrayData](GeomVertexArrayData.md)
