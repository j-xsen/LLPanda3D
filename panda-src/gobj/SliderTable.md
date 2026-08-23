# SliderTable

**Source:** `panda/src/gobj/sliderTable.h` (+ `.I`, `.cxx`)
**Inherits:** `TypedWritableReferenceCount`

Stores the full set of [VertexSlider](VertexSlider.md)s that a
`GeomVertexData` set up for CPU morph-target (blend shape) animation might
reference — parallel in structure to
[TransformTable](TransformTable.md), but keyed by **name** rather than
index, and each entry additionally records *which vertex rows* that slider
affects. Morphs are CPU-only in Panda (no GPU morph-target support), unlike
skinning which has both a CPU path (`TransformBlendTable`) and a GPU path
(`TransformTable`).

## Behavior notes

- Same register/lock (not intern) discipline as `TransformTable`:
  `register_table()` doesn't deduplicate against a global registry; it
  locks the table (`_is_registered = true`) and hooks up back-pointers
  from each contained `VertexSlider` so `mark_modified()` calls propagate.
  Mutating methods (`set_slider()`, `set_slider_rows()`, `remove_slider()`,
  `add_slider()`) are only valid on an unregistered table.
- `add_slider(slider, rows)` takes a `SparseArray` of affected row indices
  alongside the slider itself — this is how the table knows which subset
  of a `GeomVertexData`'s vertices a given morph target's delta columns
  apply to (a morph rarely touches every vertex in a mesh).
- Lookup is dual-indexed: `_sliders` (a flat `pvector<SliderDef>`, walked
  by index for `get_slider(n)`/`get_slider_rows(n)`) and
  `_sliders_by_name` (a `pmap<CPT_InternalName, SparseArray>`, consulted by
  `find_sliders(name)`/`has_slider(name)` — note `find_sliders()` returns
  the union of rows across all sliders with that name, via `SparseArray`,
  not a single slider object).
- `is_empty()` is a fast path distinct from `get_num_sliders() == 0`
  checks elsewhere in the codebase — worth using when just checking "does
  this GeomVertexData have any morph animation at all."

## API

| Method | Notes |
|---|---|
| `SliderTable()` | Empty, unregistered table. |
| `is_registered()` | Whether `register_table()` has locked this instance. |
| `static register_table(const SliderTable *)` | Lock and hook up back-pointers; no deduplication (see note above). |
| `get_num_sliders()` / `get_slider(n)` | `get_sliders()` is a `MAKE_SEQ`. |
| `get_slider_rows(n)` | `SparseArray` of rows the nth slider affects. |
| `find_sliders(const InternalName *)` | Union of rows across all sliders registered under that name. |
| `has_slider(const InternalName *)` | |
| `is_empty()` | Fast "no morph animation at all" check. |
| `set_slider(n, const VertexSlider *)` / `set_slider_rows(n, const SparseArray &)` | Unregistered only. |
| `remove_slider(n)` | Unregistered only. |
| `add_slider(const VertexSlider *, const SparseArray &rows)` | Unregistered only; returns new index. |
| `get_modified(Thread*)` | Modification sequence, updated when any contained slider changes (once registered). |
| `write(ostream&)` | Debug dump. |

## Usage

```cpp
PT(SliderTable) table = new SliderTable;
SparseArray affected_rows;
affected_rows.set_range(0, 50);  // e.g. first 50 vertices are the mouth
table->add_slider(smile_slider, affected_rows);
CPT(SliderTable) locked = SliderTable::register_table(table);
vdata->set_slider_table(locked);
```

## See also

- [VertexSlider](VertexSlider.md) — entries in this table
- [TransformTable](TransformTable.md) — structurally similar, index-keyed table for hardware skinning
- [GeomVertexData](GeomVertexData.md)
