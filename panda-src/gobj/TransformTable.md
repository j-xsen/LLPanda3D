# TransformTable

**Source:** `panda/src/gobj/transformTable.h` (+ `.I`, `.cxx`)
**Inherits:** `TypedWritableReferenceCount`

Stores the full ordered set of [VertexTransform](VertexTransform.md)s that
a `GeomVertexData` set up for **hardware matrix-palette skinning** might
reference — the GPU-side counterpart to
[TransformBlendTable](TransformBlendTable.md) (which is for CPU-computed
soft-skinning instead). Vertices reference transforms in this table by
index (typically via an indexed vertex column, per
[GeomVertexAnimationSpec](GeomVertexAnimationSpec.md)'s
`indexed_transforms` flag) rather than by name.

## Behavior notes

- **Register/lock, not intern.** `register_table()` (a static factory
  taking `const TransformTable *` and returning a `CPT(TransformTable)`)
  does **not** deduplicate against a global registry the way
  `GeomVertexFormat::register_format()` does — the header comment states
  this explicitly ("We don't actually bother adding the table object to a
  registry... there may be multiple copies of identical registered
  TransformTables. Big deal."). What registering actually does is flip
  `_is_registered` to `true` and, critically, have every referenced
  `VertexTransform` record a back-pointer to this table (via
  `VertexTransform::_tables`), so that when a transform calls
  `mark_modified()` it can push the new modified-sequence number to every
  table that depends on it. Once registered, `set_transform()`/
  `insert_transform()`/`remove_transform()` assert (mutation requires an
  unregistered, i.e. private, copy first — same COW-adjacent discipline as
  elsewhere in `gobj`, just implemented via an explicit register/unregister
  flag instead of `CopyOnWriteObject`).
- The destructor auto-unregisters if still registered, removing this
  table from every referenced transform's back-pointer set.
- `add_transform()` does **not** deduplicate — appending an already-present
  transform pointer adds a second, distinct index for it — unlike
  `TransformBlend::add_transform()`, which merges weights for a repeated
  transform.

## API

| Method | Notes |
|---|---|
| `TransformTable()` | Empty, unregistered table. |
| `is_registered()` | Whether `register_table()` has locked this instance. |
| `static register_table(const TransformTable *)` | Lock and hook up back-pointers; returns the same pointer as a `CPT`. See note above — no deduplication. |
| `get_num_transforms()` / `get_transform(n)` | `get_transforms()` is a `MAKE_SEQ`. |
| `set_transform(n, const VertexTransform *)` | Replace; unregistered only. |
| `insert_transform(n, const VertexTransform *)` | Insert at index (or append if `n` is past the end); unregistered only, no dedup. |
| `remove_transform(n)` | Remove by index; unregistered only. |
| `add_transform(const VertexTransform *)` | Append; returns new index; unregistered only. |
| `get_modified(Thread*)` | Modification sequence, updated whenever any referenced transform changes (once registered). |
| `write(ostream&)` | Debug dump. |

## Usage

```cpp
PT(TransformTable) table = new TransformTable;
size_t hip_idx = table->add_transform(hip_transform);
size_t thigh_idx = table->add_transform(thigh_transform);
CPT(TransformTable) locked = TransformTable::register_table(table);
vdata->set_transform_table(locked);
// vertex "transform_index" columns then reference hip_idx/thigh_idx
```

## See also

- [VertexTransform](VertexTransform.md) — entries in this table
- [TransformBlendTable](TransformBlendTable.md) — CPU-skinning equivalent
- [GeomVertexData](GeomVertexData.md)
