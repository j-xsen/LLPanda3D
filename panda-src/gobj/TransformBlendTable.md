# TransformBlendTable

**Source:** `panda/src/gobj/transformBlendTable.h` (+ `.I`, `.cxx`)
**Inherits:** [CopyOnWriteObject](README.md)

Per-[GeomVertexData](GeomVertexData.md) table of
[TransformBlend](TransformBlend.md)s used to compute soft-skinned/morphed
vertex positions **on the CPU**. Each vertex row references exactly one
entry in this table (by index, via a vertex column with
`Contents::C_index`); many vertices commonly share the same entry since
skinning weights are usually per-body-part rather than truly per-vertex.
The GPU-side equivalent — hardware skinning — uses
[TransformTable](TransformTable.md) instead; a `GeomVertexData` uses one or
the other depending on its
[GeomVertexAnimationSpec](GeomVertexAnimationSpec.md)'s `AnimationType`.

## Behavior notes

- Inherits `CopyOnWriteObject`, same cheap-sharing pattern as `Geom`/
  `GeomVertexData` — see [../README.md](README.md#copy-on-write-and-interning)
  — rather than being registered/interned like `TransformTable` chooses to
  be. The comment in the header explicitly notes it deliberately skips
  building a full CycleData-protected structure for the blend vector
  itself, relying on `GeomVertexData`'s own COW pointer semantics to
  guarantee the table is copied before any in-place mutation; only a small
  modified-sequence cache is kept per-pipeline-stage.
- `add_blend()`/`set_blend()` maintain a `BlendIndex` (`pmap<const
  TransformBlend*, int, IndirectLess>`) for reverse lookup — because the
  underlying `pvector<TransformBlend>` can reallocate on insertion,
  **the entire index is rebuilt from scratch (`rebuild_index()`) any time
  the vector is mutated**, not incrementally updated. This makes
  `add_blend()`/`remove_blend()` O(n) each; fine for one-time table
  construction, worth knowing before calling either in a tight per-vertex
  loop.
- `get_num_transforms()` and `get_max_simultaneous_transforms()` are
  precomputed summary stats (total distinct `VertexTransform`s referenced
  across all blends, and the largest single blend's transform count) —
  useful for the GSG to decide e.g. shader constant array sizing, even
  though this table itself is only used for CPU animation.
- `get_rows()`/`set_rows()`/`modify_rows()` hold a `SparseArray` marking
  which vertex rows are actually animated (vs. static rows sharing the
  same `GeomVertexData` but not referencing any blend) — lets
  `animate_vertices()` skip untouched rows.
- Modification tracking mirrors `TransformBlend`'s own scheme: a cached
  `_global_modified` compared against
  `VertexTransform::get_global_modified()` short-circuits a full rescan of
  every contained blend when nothing has changed process-wide.

## API

| Method | Notes |
|---|---|
| `TransformBlendTable()` | Empty table. |
| `get_num_blends()` / `get_blend(n)` | `get_blends()` is a `MAKE_SEQ`. |
| `set_blend(n, const TransformBlend&)` / `add_blend(const TransformBlend&)` | `add_blend()` returns the new entry's index. |
| `remove_blend(n)` | Remove an entry (rebuilds the index). |
| `get_num_transforms()` | Total distinct `VertexTransform`s referenced across all blends. |
| `get_max_simultaneous_transforms()` | Largest per-blend transform count in the table. |
| `get_rows()` / `set_rows(const SparseArray&)` / `modify_rows()` | Which vertex rows are animated. |
| `get_modified(Thread*)` | Modification sequence, recomputed lazily against contributing transforms. |
| `write(ostream&, indent_level)` | Debug dump of every blend. |

## Usage

```cpp
PT(TransformBlendTable) table = new TransformBlendTable;
TransformBlend blend(hip, 0.7f, thigh, 0.3f);
size_t idx = table->add_blend(blend);
vdata->set_transform_blend_table(table);
// a vertex column of Contents::C_index in vdata's format then selects
// which blend (by row value == idx) applies to each vertex
```

## See also

- [TransformBlend](TransformBlend.md) — one entry in this table
- [TransformTable](TransformTable.md) — the GPU-skinning equivalent
- [GeomVertexData](GeomVertexData.md), [GeomVertexAnimationSpec](GeomVertexAnimationSpec.md)
