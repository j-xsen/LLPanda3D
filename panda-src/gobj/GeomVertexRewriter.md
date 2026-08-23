# GeomVertexRewriter

**Source:** `panda/src/gobj/geomVertexRewriter.h` (+ `.I`, `.cxx`)
**Inherits:** [GeomVertexWriter](GeomVertexWriter.md) > ... , [GeomVertexReader](GeomVertexReader.md) > ... (multiple inheritance — combines both directly)

Combines a [GeomVertexReader](GeomVertexReader.md) and a
[GeomVertexWriter](GeomVertexWriter.md) into one object for making a single
pass over a `GeomVertexData` column, reading and modifying rows in place
(e.g. transforming existing vertex positions). It inherits from *both*
`GeomVertexWriter` and `GeomVertexReader` directly (multiple inheritance,
not a wrapper) — reader-side and writer-side methods are both available
on the same instance.

## Behavior notes

- Doesn't offer a real performance advantage over separately-constructed
  reader + writer — its value is correctness: it internally manages
  construction order (writer before reader, per the reference-count/
  reallocation caveat documented on both base classes) so callers can't get
  it wrong. Prefer this over a manual pair whenever both read and write
  access to the same column is needed.
- Method sets from both bases are present but several are re-declared here
  to disambiguate the multiple-inheritance overlap (`set_column`, `clear`,
  `has_column`, `get_array`, `get_column`, `set_row`/`set_row_unsafe`,
  `get_start_row`, `is_at_end`, `output` all exist on both `Reader` and
  `Writer`) — calling any of these operates on both the read and write
  cursor state simultaneously (they stay in sync), rather than needing to
  call the reader's and writer's versions separately.
- `get_data*()` (from `GeomVertexReader`) and `set_data*()`/`add_data*()`
  (from `GeomVertexWriter`) are both usable — a typical read-modify-write
  loop calls `get_data3f()` then `set_data3f()` at the same row.
- Same underlying-buffer lifetime caveats as the two base classes apply
  (don't hold across reallocation-triggering operations, don't keep
  long-term).

## API

Constructors mirror `GeomVertexReader`/`GeomVertexWriter` (data object,
optional column name/index, optional `Thread*`). All other methods are
inherited from [GeomVertexReader](GeomVertexReader.md) and
[GeomVertexWriter](GeomVertexWriter.md) — see those docs for the full
method tables; this class adds no new members beyond disambiguating
overrides.

| Method | Notes |
|---|---|
| `GeomVertexRewriter(Thread *thread = current)` | Invalid rewriter; must be assigned before use. |
| `GeomVertexRewriter(GeomVertexData *data, [CPT_InternalName name,] Thread *thread = current)` | Bound to a `GeomVertexData`, optionally to a named column. |
| `GeomVertexRewriter(GeomVertexArrayData *array, [int column,] Thread *thread = current)` | Scoped to one array only. |
| (reader methods) | `get_data1f()`..`get_data4i()`, `get_matrix3f()`/`get_matrix4f()` (+d), `get_read_row()`, `get_force()`/`set_force()` — see [GeomVertexReader](GeomVertexReader.md). |
| (writer methods) | `set_data1f()`..`add_data4i()`, `set_matrix3f()`/`set_matrix4f()` (+d), `get_write_row()`, `reserve_num_rows()` — see [GeomVertexWriter](GeomVertexWriter.md). |

## Usage

```cpp
GeomVertexRewriter vertex(vdata, InternalName::get_vertex());
while (!vertex.is_at_end()) {
  LVecBase3f v = vertex.get_data3f();
  vertex.set_data3f(v * 2.0f);  // scale every vertex in place
}
```

## See also

- [GeomVertexReader](GeomVertexReader.md), [GeomVertexWriter](GeomVertexWriter.md) — base classes, full API detail
- [GeomVertexData](GeomVertexData.md)
