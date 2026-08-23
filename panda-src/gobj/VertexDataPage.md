# VertexDataPage

**Source:** `panda/src/gobj/vertexDataPage.h` (+ `.I`, `.cxx`)
**Inherits:** SimpleAllocator, SimpleLruPage (see [SimpleAllocator.md](SimpleAllocator.md), [SimpleLru.md](SimpleLru.md)) **Inherited by:** (none)

A block of bytes holding one or more [`VertexDataBlock`](VertexDataBlock.md)s, and the unit of eviction for the module README's ["vertex data disk paging"](README.md#residency-tracking-lrus-and-allocators) subsystem. `VertexDataPage` is the class that literally bridges the two caching subsystems described in the README — it's simultaneously a `SimpleAllocator` (sub-allocating its byte range into individual `VertexDataBlock`s, one per `GeomVertexArrayData` that lands on it) *and* a `SimpleLruPage` (so it can be tracked and evicted by a global recency-ordered `SimpleLru`, independently of GPU residency).

## Behavior notes

- **Three-state residency, tracked per-page:** `RC_resident` (page data sitting in normal system RAM, directly accessible), `RC_compressed` (in-memory zlib-deflated, not directly accessible — must decompress to read), `RC_disk` (written out to the shared [`VertexDataSaveFile`](VertexDataSaveFile.md) and freed from RAM entirely). One **global `SimpleLru` per state** (`_resident_lru`, `_compressed_lru`, `_disk_lru`, indexed via `get_global_lru(RamClass)`) plus one more, `_pending_lru`, for pages currently queued for a background state transition — five LRUs total, and a page moves between them as its state changes (`set_ram_class()` calls `mark_used_lru(_global_lru[rclass])`, physically relocating the page's `SimpleLruPage` node onto the new LRU).
- **State transitions happen asynchronously, on background `PageThread`s.** `request_resident()`/`request_ram_class()` don't block by default — they set `_pending_ram_class` and hand the page to a `PageThreadManager` (a small thread pool, `PageThread` subclassing `Thread`), which eventually calls `make_resident()`/`make_compressed()`/`make_disk()` off-thread and only then updates `_ram_class` to match. `get_page_data(force=false)` can return `nullptr` if the page isn't resident yet (having merely *requested* residency); `get_page_data(force=true)` instead calls `make_resident_now()`, blocking synchronously until the data is available — the two call sites represent "I can wait a frame" vs. "I need this data right now."
- **Compression is chunked, not whole-page.** While deflating, the page builds a temporary linked list of small `DeflatePage` nodes (`deflate_page_size = 1024` bytes each, pool-allocated via `ALLOC_DELETED_CHAIN`) rather than compressing into one contiguous buffer — presumably to avoid needing to know the compressed size up front.
- **`operator<` orders by available contiguous space, not identity** — used by [`VertexDataBook`](VertexDataBook.md)'s `pset<VertexDataPage*, IndirectLess<...>>` to keep pages ordered so allocation can efficiently find "the smallest page that still has room," a first-fit-by-best-remaining-space strategy. Ties break on pointer identity purely for lookup determinism.
- **`save_to_disk()` is a soft/lazy operation:** it writes the page's current bytes to the save file but does *not* evict the page from RAM or touch its LRU status — the point (per the doc comment) is that if the page later *does* get evicted from memory unmodified, it won't need to redo the disk write, since the on-disk copy is already known-current.
- **Global thread-pool controls are static/process-wide**, not per-page: `stop_threads()`/`flush_threads()`/`get_num_threads()`/`get_num_pending_reads()`/`get_num_pending_writes()` all operate on the one shared `PageThreadManager`. Thread count and behavior are governed by the `vertex-data-page-threads` config var (README's config table).
- `_book_size` (used by `operator<`) is kept in sync via `adjust_book_size()`, called whenever the ram class changes — since a non-resident page effectively has zero "available space" from the book's allocation-placement perspective even if its underlying `SimpleAllocator` free space is nonzero.

## API

| Signature | Notes |
|---|---|
| `RamClass get_ram_class() const` / `get_pending_ram_class() const` | Current state / in-flight target state (differ while a background transition is queued). |
| `void request_resident()` | Asynchronously request `RC_resident`; doesn't block. |
| `unsigned char *get_page_data(bool force)` | Access the raw bytes; `force=true` blocks until resident, `force=false` may return `nullptr`. |
| `VertexDataBlock *alloc(size_t size)` | Sub-allocate a block from this page (delegates to `SimpleAllocator`). |
| `VertexDataBlock *get_first_block() const` | Walk the page's allocated blocks. |
| `VertexDataBook *get_book() const` | Owning book. |
| `static SimpleLru *get_global_lru(RamClass rclass)` | The shared LRU for a given residency state. |
| `static VertexDataSaveFile *get_save_file()` | The shared on-disk backing file (lazily created). |
| `bool save_to_disk()` | Write current bytes to disk without evicting/changing LRU status. |
| `static int get_num_threads()` / `get_num_pending_reads()` / `get_num_pending_writes()` | Background paging-thread-pool stats. |
| `static void stop_threads()` / `flush_threads()` | Shut down / drain the paging thread pool. |

## Usage

Not constructed directly by application code — see [`VertexDataBook::alloc()`](VertexDataBook.md), which creates pages on demand as needed to satisfy allocation requests. A `GeomVertexArrayData`'s buffer ultimately lands in a `VertexDataBlock` on one of these pages when it's paged out of its "independent" resident state (see [VertexDataBuffer](VertexDataBuffer.md)).

## See also

- [VertexDataBlock](VertexDataBlock.md) — the sub-allocations living on a page
- [VertexDataBook](VertexDataBook.md) — the collection of pages this belongs to
- [VertexDataSaveFile](VertexDataSaveFile.md) — the on-disk backing store for `RC_disk` pages
- [SimpleAllocator](SimpleAllocator.md), [SimpleLru](SimpleLru.md) — the two base-class subsystems this bridges
