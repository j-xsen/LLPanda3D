# GeomVertexArrayData

**Source:** `panda/src/gobj/geomVertexArrayData.h` (+ `.I`, `.cxx`; `geomVertexArrayData_ext.h/.cxx` excluded — Python buffer-protocol glue)
**Inherits:** `CopyOnWriteObject`, `SimpleLruPage`, `GeomEnums` (see [gobj README](README.md#copy-on-write-and-interning) / [shared enums](README.md#shared-enums-geomenums))
**Inherited by:** (none)

The data for one physical array of a [GeomVertexData](GeomVertexData.md) —
DirectX calls this concept a "stream," and it closely corresponds to one
GPU vertex buffer. It's just a flat block of bytes (`VertexDataBuffer`)
whose interpretation is defined entirely by its
[GeomVertexArrayFormat](GeomVertexArrayFormat.md). Application code should
not manipulate this directly — go through `GeomVertexData` and
[GeomVertexReader/Writer/Rewriter](GeomVertexReader.md) instead; this class
exists mainly as the unit of copy-on-write sharing, GPU preparation, and
LRU residency tracking.

## Behavior notes

- **Copy-on-write, same pattern as `Geom`.** See
  [gobj README](README.md#copy-on-write-and-interning). A `GeomVertexData`
  holds each of its arrays as a `COWPT(GeomVertexArrayData)`; reading
  (`get_array()`) returns a read pointer, writing (`modify_array()`)
  triggers a copy only if shared.
- **`SimpleLruPage` inheritance is for disk-paging eligibility, not GPU
  residency** — don't confuse it with the `AdaptiveLruPage` used by
  `TextureContext`/`VertexBufferContext`/`IndexBufferContext` for GPU
  memory. This class's `evict_lru()` override calls `dequeue_lru()` then
  `cdata->_buffer.page_out(_book)` — i.e. LRU eviction here means paging
  the raw byte buffer out to a `VertexDataSaveFile` via the shared
  `VertexDataBook`, not releasing a GPU resource. See
  [gobj README](README.md#residency-tracking-lrus-and-allocators) and
  [VertexDataPage](VertexDataPage.md).
- **GPU preparation is a separate, independent lifecycle** from the LRU
  paging above: `prepare(pgo)` enqueues the array for upload at the start
  of next frame; `prepare_now(pgo, gsg)` immediately creates (or returns
  the existing) [VertexBufferContext](VertexBufferContext.md), tracked in
  a private `map<PreparedGraphicsObjects*, VertexBufferContext*>` since the
  same array can be prepared on multiple GSGs at once (e.g. multi-window
  apps). `release(pgo)` frees just that GSG's context (or dequeues a
  pending `prepare()` if it hadn't uploaded yet); `release_all()` frees
  every context. See
  [gobj README](README.md#preparedgraphicsobjects--context-handshake).
- **Read/write locking via `GeomVertexArrayDataHandle`:** `get_handle()`/
  `modify_handle()` return a `GeomVertexArrayDataHandle`, which holds
  `CData::_rw_lock` (a `ReMutex`) for its lifetime — the underlying data is
  locked as long as any handle exists. The lock is reentrant (a thread may
  lock it multiple times), but only one thread may hold it at a time;
  other threads block. `GeomVertexReader`/`Writer`/`Rewriter` acquire
  handles internally, so application code normally never sees this
  directly.
- Two static class-wide LRUs (`_independent_lru`, `_small_lru`, plus the
  `VertexDataPage`-owned resident/compressed LRUs) and one shared
  `VertexDataBook` (`_book`) back every `GeomVertexArrayData` instance —
  `lru_epoch()` (called once per frame by the engine) ticks all of them to
  consider evictions.

## API

| Method | Notes |
|---|---|
| `GeomVertexArrayData(array_format, usage_hint)` | — |
| `const GeomVertexArrayFormat *get_array_format() const` | — |
| `UsageHint get_usage_hint() const` / `set_usage_hint(hint)` | — |
| `bool has_column(name) const` | — |
| `int get_num_rows() const` / `set_num_rows(n)` / `unclean_set_num_rows(n)` / `reserve_num_rows(n)` / `clear_rows()` | Same semantics as the `GeomVertexData` equivalents (see [GeomVertexData](GeomVertexData.md)). |
| `size_t get_data_size_bytes() const` / `UpdateSeq get_modified() const` | — |
| `bool request_resident(thread) const` | Returns false (and triggers async fault-in) if paged out. |
| `CPT(GeomVertexArrayDataHandle) get_handle(thread) const` / `PT(...) modify_handle(thread)` | Explicit locked-handle access; see Behavior notes. |
| `void prepare(pgo)` / `bool is_prepared(pgo) const` / `VertexBufferContext *prepare_now(pgo, gsg)` / `bool release(pgo)` / `int release_all()` | GPU preparation lifecycle. |
| `static SimpleLru *get_independent_lru()` / `get_small_lru()` / `static void lru_epoch()` / `static VertexDataBook &get_book()` | Class-wide shared state. |
| `int compare_to(other) const` | — |
| `void output(out) const` / `write(out, indent=0) const` | — |

`GeomVertexArrayDataHandle` (same header) is a `ReferenceCount, GeomEnums`
lock-holding accessor returned by `get_handle()`/`modify_handle()`; its
`copy_data_from()`/`get_data()`/`set_data()`/`get_subdata()`/`set_subdata()`
are the actual raw-byte read/write entry points `GeomVertexReader`/`Writer`
use internally.

## Usage

```cpp
// Normally accessed only through GeomVertexData:
CPT(GeomVertexArrayData) arr = vdata->get_array(0);
nout << "array 0: " << arr->get_data_size_bytes() << " bytes, "
     << arr->get_num_rows() << " rows\n";
```

## See also

- [gobj README](README.md) — COW pattern, LRU/residency tracking, GPU handshake
- [GeomVertexData](GeomVertexData.md) — owns a list of these
- [GeomVertexArrayFormat](GeomVertexArrayFormat.md) — this array's layout
- [VertexBufferContext](VertexBufferContext.md), [PreparedGraphicsObjects](PreparedGraphicsObjects.md)
- [VertexDataPage](VertexDataPage.md), [SimpleLru](SimpleLru.md) — disk-paging LRU machinery
