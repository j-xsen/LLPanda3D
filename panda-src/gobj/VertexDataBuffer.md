# VertexDataBuffer

**Source:** `panda/src/gobj/vertexDataBuffer.h` (+ `.I`, `.cxx`)
**Inherits:** (none) **Inherited by:** (none)

The actual raw-byte storage backing a [`GeomVertexArrayData`](GeomVertexArrayData.md), and the class that decides — per-buffer — whether that storage currently lives directly in memory or has been handed off to the page/disk system described in the module README's [residency tracking](README.md#residency-tracking-lrus-and-allocators) section.

## Behavior notes (from the header's own design comment — worth quoting near-verbatim, it's precise)

A buffer is always in exactly one of two states:

- **Independent** — the buffer's memory is resident and owned directly by the `VertexDataBuffer` object itself, in `_resident_data`. `_reserved_size` may exceed `_size` (over-allocated capacity, amortizing future growth). This is the freely-read/write state.
- **Paged** — the buffer's memory is owned by a [`VertexDataBlock`](VertexDataBlock.md) instead. That block's *page* might itself be resident, compressed, or on disk (see [VertexDataPage](VertexDataPage.md)'s `RamClass`). Even when the page is resident, memory reached this way is **read-only** — `_reserved_size == _size` always holds in this state (no spare growth capacity, since it's not meant to be written in place).

`VertexDataBuffer`s always start **independent**. They get pushed into **paged** state when their owning `GeomVertexArrayData` is evicted from a global `_independent_lru` (the recency LRU covering all currently-independent buffers — distinct from the page-level LRUs `VertexDataPage` uses). They move back to **independent** automatically the moment they're modified — `get_write_pointer()` or `realloc()`-family calls trigger `do_page_in()` first if currently paged, since paged memory is read-only. The intent (again, from the header): keep hot/frequently-modified vertex data cheaply accessible as ordinary resident memory, while cold/static data gets swept together onto pages that can be compressed or written to disk as a unit.

## Behavior notes (additional, from the API shape)

- `get_read_pointer(bool force)` is `const` and returns read-only access without forcing a page-in transition — appropriate for the common "just render this" path; `get_write_pointer()` is non-const and implicitly pages the buffer back to independent state if needed, since writing paged (read-only) memory isn't allowed.
- `clean_realloc()` vs. `unclean_realloc()`: the "clean" variant presumably preserves existing content up to the new size (a real realloc), while "unclean" just grabs a new buffer of the requested reserved size without guaranteeing old content survives — check call sites/the `.cxx` if the distinction matters for a specific use (naming strongly suggests clean=preserving, unclean=don't-care, but confirm before relying on it for anything content-sensitive).
- `page_out(VertexDataBook &book)` is the explicit/manual push-to-paged-state entry point, taking a specific `VertexDataBook` to allocate from — used when something other than pure LRU pressure wants to page a buffer out immediately (e.g. explicit "I know I won't touch this again soon" hints).
- `swap()` exchanges two buffers' internal state (pointer/size fields) without copying data — used for efficient buffer content replacement (e.g. reformatting) without a full copy.
- `_lock` is a `LightMutex` — lighter-weight than the `Mutex`/`ReMutex` used elsewhere in this fork, suggesting buffer-level locking is meant to be cheap/frequent (every read/write pointer access takes it).
- Both pointer accessors are `RETURNS_ALIGNED(MEMORY_HOOK_ALIGNMENT)` — the returned pointers are guaranteed aligned per Panda's configured memory alignment (relevant for SIMD access to vertex data).

## API

| Signature | Notes |
|---|---|
| `VertexDataBuffer()` / `VertexDataBuffer(size_t size)` | Starts in independent state. |
| `const unsigned char *get_read_pointer(bool force) const` | Read access; `force` controls blocking-vs-`nullptr` if currently paged to a non-resident page. |
| `unsigned char *get_write_pointer()` | Write access; implicitly pages back in to independent state first if needed. |
| `size_t get_size() const` / `get_reserved_size() const` | Logical size vs. allocated capacity. |
| `void set_size(size_t size)` | Shrink/grow the logical size within reserved capacity (or trigger reallocation). |
| `void clean_realloc(size_t reserved_size)` / `unclean_realloc(size_t reserved_size)` | Resize reserved capacity, preserving vs. not-necessarily-preserving content. |
| `void clear()` | Reset to empty. |
| `void page_out(VertexDataBook &book)` | Explicitly transition to paged state, allocating from `book`. |
| `void swap(VertexDataBuffer &other)` | Exchange internal state cheaply. |

## Usage

Owned internally by `GeomVertexArrayData` — not typically constructed standalone by application code, but the read/write pointer distinction matters when working with `GeomVertexArrayData`'s buffer directly:

```cpp
// read-only, may page in but won't force a copy-on-write:
const unsigned char *ro = buffer.get_read_pointer(true);
// write access, forces independent (resident, writable) state:
unsigned char *rw = buffer.get_write_pointer();
```

## See also

- [VertexDataBlock](VertexDataBlock.md), [VertexDataPage](VertexDataPage.md), [VertexDataBook](VertexDataBook.md) — the paged-state machinery this buffer transitions into
- [GeomVertexArrayData](GeomVertexArrayData.md) — the owner of a `VertexDataBuffer`
