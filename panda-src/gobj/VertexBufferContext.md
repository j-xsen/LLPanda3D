# VertexBufferContext

**Source:** `panda/src/gobj/vertexBufferContext.h` (+ `.I`, `.cxx`)
**Inherits:** [BufferContext](BufferContext.md), AdaptiveLruPage (see [AdaptiveLru.md](AdaptiveLru.md)) **Inherited by:** (none)

GSG-side handle for an uploaded [`GeomVertexArrayData`](GeomVertexArrayData.md) — e.g. an OpenGL vertex buffer object or a DirectX vertex buffer. Structurally identical to [`IndexBufferContext`](IndexBufferContext.md) except it wraps vertex data instead of index data, and compares against a `GeomVertexArrayDataHandle` instead of a `GeomPrimitivePipelineReader`.

## Behavior notes

Same design as `IndexBufferContext` throughout — see that doc for the shared reasoning (dual residency+LRU size tracking, `mark_loaded()`/`mark_unloaded()` semantics). The only real differences:
- Constructed against `pgo->_vbuffer_residency` (a separate `BufferResidencyTracker` from `_ibuffer_residency`, so PStats reports vertex-buffer and index-buffer memory usage as distinct categories).
- `get_data()` returns `GeomVertexArrayData*`; the change-detection methods (`changed_size()`, `changed_usage_hint()`, `was_modified()`) and `mark_loaded()` all take a `const GeomVertexArrayDataHandle*` rather than a `GeomPrimitivePipelineReader*`.
- `output()` prints the underlying `GeomVertexArrayData`'s `operator<<` output plus byte size.

## API

| Signature | Notes |
|---|---|
| `VertexBufferContext(PreparedGraphicsObjects *pgo, GeomVertexArrayData *data)` | Registers on `pgo`'s vertex-buffer residency tracker. |
| `GeomVertexArrayData *get_data() const` | The array this context backs. |
| `bool changed_size(const GeomVertexArrayDataHandle *reader) const` | True if size differs from what was last uploaded. |
| `bool changed_usage_hint(const GeomVertexArrayDataHandle *reader) const` | True if `UsageHint` differs from last upload. |
| `bool was_modified(const GeomVertexArrayDataHandle *reader) const` | True if the array's modified-sequence has advanced since last upload. |
| `void mark_loaded(const GeomVertexArrayDataHandle *reader)` | Call after a successful GPU upload; syncs cached state, marks resident. |
| `void mark_unloaded()` | Call after the GSG evicts this buffer; marks stale/nonresident. |

## Usage

```cpp
VertexBufferContext *vbc = array_data->prepare_now(prepared_objects, gsg);
if (vbc->was_modified(handle) || vbc->changed_size(handle)) {
  // re-upload vertex data to the GPU ...
  vbc->mark_loaded(handle);
}
```

## See also

- [IndexBufferContext](IndexBufferContext.md) — the index-data counterpart, structurally identical
- [BufferContext](BufferContext.md), [AdaptiveLru](AdaptiveLru.md) — base classes providing residency and eviction tracking
- [GeomVertexArrayData](GeomVertexArrayData.md) — the object this context backs
- [PreparedGraphicsObjects](PreparedGraphicsObjects.md) — owns and dispatches these
