# IndexBufferContext

**Source:** `panda/src/gobj/indexBufferContext.h` (+ `.I`, `.cxx`)
**Inherits:** [BufferContext](BufferContext.md), AdaptiveLruPage (see [AdaptiveLru.md](AdaptiveLru.md)) **Inherited by:** (none)

GSG-side handle for an uploaded [`GeomPrimitive`](GeomPrimitive.md)'s index array — e.g. an OpenGL element buffer object or a DirectX index buffer. One of the three concrete `BufferContext` subclasses, structurally identical to [`VertexBufferContext`](VertexBufferContext.md) except it wraps a `GeomPrimitive` (index data) instead of a `GeomVertexArrayData` (vertex data).

## Behavior notes

- Constructed against `pgo->_ibuffer_residency` — every `IndexBufferContext` for one `PreparedGraphicsObjects` shares that one [`BufferResidencyTracker`](BufferResidencyTracker.md), keeping index-buffer memory reporting separate from vertex-buffer/texture reporting in PStats.
- `update_data_size_bytes()` updates *both* the `BufferContext` byte total (for `BufferContextChain`/PStats) and `AdaptiveLruPage::set_lru_size()` (for eviction-cost weighting in the [`AdaptiveLru`](AdaptiveLru.md)) — the two caching subsystems described in the module README both need to know the object's size, from one call site.
- `changed_size()`/`changed_usage_hint()`/`was_modified()` compare cached state against a fresh `GeomPrimitivePipelineReader` snapshot — this is how a GSG backend decides, cheaply, whether it needs to re-upload the index data or can reuse what's already on the card.
- `mark_loaded()` is the single call a GSG backend makes after a successful upload: it updates the cached size/modified-sequence/usage-hint from the reader, and unconditionally calls `set_resident(true)` — the assumption baked in is "if we just finished uploading it, it's resident."
- `mark_unloaded()` resets the modified-sequence to `UpdateSeq::old()` (forcing the next `was_modified()` check to report stale/needs-reupload) and calls `set_resident(false)`.
- `output()` prints the underlying `GeomPrimitive`'s own `operator<<` output plus the byte size — so debug/PStats output shows meaningful primitive info, not just a pointer.

## API

| Signature | Notes |
|---|---|
| `IndexBufferContext(PreparedGraphicsObjects *pgo, GeomPrimitive *data)` | Registers on `pgo`'s index-buffer residency tracker. |
| `GeomPrimitive *get_data() const` | The primitive this context backs. |
| `bool changed_size(const GeomPrimitivePipelineReader *reader) const` | True if index array size differs from what was last uploaded. |
| `bool changed_usage_hint(const GeomPrimitivePipelineReader *reader) const` | True if `UsageHint` differs from last upload. |
| `bool was_modified(const GeomPrimitivePipelineReader *reader) const` | True if the primitive's modified-sequence has advanced since last upload. |
| `void mark_loaded(const GeomPrimitivePipelineReader *reader)` | Call after a successful GPU upload; syncs cached state, marks resident. |
| `void mark_unloaded()` | Call after the GSG evicts this buffer; marks stale/nonresident. |

## Usage

Written/read by GSG backend code, not typical application code:

```cpp
IndexBufferContext *ibc = prim->prepare_now(prepared_objects, gsg);
if (ibc->was_modified(reader) || ibc->changed_size(reader)) {
  // re-upload index data to the GPU ...
  ibc->mark_loaded(reader);
}
```

## See also

- [VertexBufferContext](VertexBufferContext.md) — the vertex-data counterpart, structurally identical
- [BufferContext](BufferContext.md), [AdaptiveLru](AdaptiveLru.md) — base classes providing residency and eviction tracking
- [GeomPrimitive](GeomPrimitive.md) — the object this context backs
- [PreparedGraphicsObjects](PreparedGraphicsObjects.md) — owns and dispatches these
