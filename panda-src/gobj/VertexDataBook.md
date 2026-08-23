# VertexDataBook

**Source:** `panda/src/gobj/vertexDataBook.h` (+ `.I`, `.cxx`)
**Inherits:** (none) **Inherited by:** (none)

A collection of [`VertexDataPage`](VertexDataPage.md)s that can be allocated from as a unit — the top-level entry point application-adjacent code uses to get a [`VertexDataBlock`](VertexDataBlock.md) without worrying about which specific page it lands on. Creates new pages on demand as existing ones fill up.

## Behavior notes

- **Allocation strategy: best-fit among existing pages, else grow.** `do_alloc(size)` scans `_pages` (a `pset<VertexDataPage*, IndirectLess<VertexDataPage>>`, kept ordered by `VertexDataPage::operator<` — i.e. by available contiguous space) looking for a page that can satisfy the request; if none can, it creates a new page (`create_new_page()`) sized to at least fit the request (rounded up per the book's `_block_size` granularity). Because `IndirectLess` orders pages by remaining space, the search is efficient rather than a linear scan of all pages every time.
- `reorder_page()` is called (by `VertexDataPage`, a `friend`) whenever a page's state changes such that its position in the ordered `_pages` set would be stale — since `operator<` depends on mutable state (`_book_size`), the set has to be explicitly re-sorted (remove + reinsert) after any such change rather than relying on `std::set`'s normal invariant, which assumes keys don't mutate in place.
- `count_total_page_size()`/`count_allocated_size()` come in both an all-pages overload and a `RamClass`-filtered overload — letting a caller ask e.g. "how many bytes are currently paged to disk across this book" for diagnostics/PStats purposes.
- `save_to_disk()` at the book level just calls `save_to_disk()` on every contained page — a bulk "flush everything to disk now" operation, e.g. useful before an expected memory-pressure event or at shutdown.
- `_lock` (a `Mutex`) protects `_pages` itself; individual page contention is handled by each `VertexDataPage`'s own lock.

## API

| Signature | Notes |
|---|---|
| `explicit VertexDataBook(size_t block_size)` | `block_size` sets the page-growth/rounding granularity. |
| `VertexDataBlock *alloc(size_t size)` | Allocate a block, reusing an existing page if one fits, else creating a new page. |
| `size_t get_num_pages() const` | Current page count. |
| `size_t count_total_page_size() const` / `count_total_page_size(RamClass)` | Total bytes across all pages / pages in a given residency state. |
| `size_t count_allocated_size() const` / `count_allocated_size(RamClass)` | Bytes actually allocated (vs. page capacity) overall / per state. |
| `void save_to_disk()` | Write every page's current data to the shared save file, without evicting. |

## Usage

```cpp
VertexDataBook book(4096);  // block_size granularity
PT(VertexDataBlock) block = book.alloc(needed_bytes);
```

## See also

- [VertexDataPage](VertexDataPage.md) — the pages this book manages and creates on demand
- [VertexDataBlock](VertexDataBlock.md) — the allocation unit returned by `alloc()`
