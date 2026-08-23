# TransformBlend

**Source:** `panda/src/gobj/transformBlend.h` (+ `.I`, `.cxx`)
**Inherits:** (none — standalone value type)

One vertex's CPU-soft-skinning "recipe": a set of `(VertexTransform, weight)`
pairs (link [VertexTransform](VertexTransform.md)) that, blended together,
produce the net transform matrix to apply to that vertex. A
[TransformBlendTable](TransformBlendTable.md) holds one `TransformBlend`
per unique blend combination, shared by however many vertices use that
exact combination.

## Behavior notes

- Entries are stored in an `ov_set` (ordered vector-set) keyed by
  `VertexTransform` pointer, so a given transform appears at most once per
  blend; `add_transform()` on an already-present transform *increases* its
  existing weight rather than adding a duplicate entry, and weights that
  net out to (near) zero cause the entry to be dropped entirely.
- `limit_transforms(max)` repeatedly removes the single
  least-weighted entry until at most `max` remain — an O(n²) linear scan
  per removal, fine for the small counts (typically ≤4) real skinning uses.
- `normalize_weights()` rescales all weights to sum to 1.0; not called
  automatically — call it explicitly after building up a blend by hand.
- **Result caching:** the blended `LMatrix4` result is memoized per-thread
  in a `PipelineCycler`-protected `CData`, keyed by a max of the
  contributing `VertexTransform`s' `get_modified()` sequence numbers (all
  transforms share one global monotonic sequence via
  `VertexTransform::get_global_modified()`, so comparing a single number is
  enough to know the cache is stale). `update_blend()` triggers a
  recompute-if-stale; `get_blend()` reads the cached matrix.
  `transform_point()`/`transform_vector()` are convenience wrappers that
  call `update_blend()` + apply the matrix in one step, in both float and
  double flavors.
- `compare_to()` defines a total ordering (by entry count, then by
  transform pointer, then by weight) used so that `TransformBlendTable`
  can deduplicate identical blends via `IndirectLess`.
- Serialization stores each `VertexTransform` as a `BamWriter` pointer
  (resolved in `complete_pointers()`) plus its weight; after pointers are
  resolved, entries are re-sorted since insertion order during `fillin()`
  isn't guaranteed to match the set's ordering.

## API

| Method | Notes |
|---|---|
| `TransformBlend()` | Empty blend. |
| `TransformBlend(t0, w0, [t1, w1, [t2, w2, [t3, w3]]])` | Up to 4 initial (transform, weight) pairs. |
| `add_transform(const VertexTransform *, PN_stdfloat weight)` | Add, or accumulate weight if already present. |
| `remove_transform(const VertexTransform *)` | Drop an entry. |
| `limit_transforms(int max_transforms)` | Trim to at most N entries, dropping least-weighted first. |
| `normalize_weights()` | Rescale weights to sum to 1.0. |
| `has_transform(const VertexTransform *)` / `get_weight(const VertexTransform *)` | Lookup by transform. |
| `get_num_transforms()` / `get_transform(n)` / `get_weight(n)` | Index-based iteration; `get_transforms()` is a `MAKE_SEQ`. |
| `set_transform(n, ...)` / `set_weight(n, ...)` / `remove_transform(n)` | Index-based mutation. |
| `update_blend(Thread*)` | Recompute the cached result if any contributing transform changed. |
| `get_blend(LMatrix4 &result, Thread*)` | Fetch the (already-updated) cached blend matrix. |
| `transform_point(LPoint3/4f/d &, Thread*)`, `transform_vector(LVector3f/d &, Thread*)` | Update + apply in one call. |
| `get_modified(Thread*)` | Cached blend's modification sequence. |
| `compare_to()`, `operator <`/`==`/`!=` | Total ordering for deduplication. |
| `output(ostream&)` / `write(ostream&, indent)` | Debug dump; `write()` also prints each transform's matrix and the final blended result. |

## Usage

```cpp
TransformBlend blend;
blend.add_transform(hip_transform, 0.7f);
blend.add_transform(thigh_transform, 0.3f);
blend.normalize_weights();
size_t n = blend_table->add_blend(blend);
// assign row n's vertex index in GeomVertexData to reference this blend
```

## See also

- [TransformBlendTable](TransformBlendTable.md) — owns a set of these, one per vertex
- [VertexTransform](VertexTransform.md) — the individual transform sources being blended
