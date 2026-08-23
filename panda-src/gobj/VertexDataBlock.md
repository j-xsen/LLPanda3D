# VertexDataBlock

**Source:** `panda/src/gobj/vertexDataBlock.h` (+ `.I`, `.cxx`)
**Inherits:** SimpleAllocatorBlock, ReferenceCount (see [SimpleAllocator.md](SimpleAllocator.md)) **Inherited by:** (none)

One sub-allocated byte range within a [`VertexDataPage`](VertexDataPage.md) — the actual raw storage for one [`GeomVertexArrayData`](GeomVertexArrayData.md)'s bytes once it's been paged out of its `VertexDataBuffer`'s "independent" resident state (see [VertexDataBuffer](VertexDataBuffer.md) for that state machine). Reference-counted (unlike its parent page, which isn't) since a block's lifetime is tied to whatever `GeomVertexArrayData`/`VertexDataBuffer` currently owns it, independent of the page's own lifetime.

## Behavior notes

- Protected constructor — only [`VertexDataPage`](VertexDataPage.md) (a `friend`) creates these, via its `alloc()`.
- `get_pointer(bool force)` delegates to the owning page's data access — if the page isn't resident, `force` controls the same blocking-vs-`nullptr` tradeoff as `VertexDataPage::get_page_data(force)`. This is the actual read/write access point application-adjacent code (specifically `VertexDataBuffer::do_page_in()`) uses once data has been paged.
- `get_next_block()` walks the intrusive allocation-order list within the owning page (same mechanism as `SimpleAllocatorBlock`'s general "walk all blocks on this allocator" support).

## API

| Signature | Notes |
|---|---|
| `VertexDataPage *get_page() const` | The page this block's bytes live on. |
| `VertexDataBlock *get_next_block() const` | Next block on the same page, in allocation order. |
| `unsigned char *get_pointer(bool force) const` | Access the bytes; `force=true` blocks until the owning page is resident. |

## Usage

Not constructed directly — obtained from `VertexDataPage::alloc()`, and typically only encountered indirectly via `VertexDataBuffer`'s paging logic:

```cpp
PT(VertexDataBlock) block = page->alloc(size);
unsigned char *data = block->get_pointer(true);  // blocks until resident
```

## See also

- [VertexDataPage](VertexDataPage.md) — owning page and allocator
- [VertexDataBuffer](VertexDataBuffer.md) — the higher-level object that transitions into "paged" state by acquiring one of these
- [SimpleAllocator](SimpleAllocator.md) — base allocation mechanism (`SimpleAllocatorBlock`)
