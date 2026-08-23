# SimpleAllocator

**Source:** `panda/src/gobj/simpleAllocator.h` (+ `.I`, `.cxx`)
**Inherits:** LinkedListNode **Inherited by:** [VertexDataPage](VertexDataPage.md), [VertexDataSaveFile](VertexDataSaveFile.md)

A minimal first-fit block allocator over a range of nonnegative integers (byte offsets, in every actual use in this module) up to a specified upper limit. This is the generic allocation mechanism underlying both the in-memory vertex-data-paging system (`VertexDataPage`) and the on-disk save file (`VertexDataSaveFile`) described in the module README's [residency tracking](README.md#residency-tracking-lrus-and-allocators) section — see that section for how it relates to the *separate* `SimpleLru`/`AdaptiveLru` eviction subsystem (a `SimpleAllocator` has no eviction policy of its own; it just tracks what ranges are free).

## Behavior notes

- **No merging needed, by construction.** The allocator's own doc comment explains the trick: only *allocated* blocks are stored, as a sorted linked-list chain (`LinkedListNode`-based); free space is *implicit* — it's whatever byte ranges fall between consecutive allocated blocks (or before the first / after the last). This means freeing a block is just "remove it from the chain," with no adjacent-free-block-merging bookkeeping required, at the cost of having to walk the allocated-block chain to find a first-fit free gap when allocating.
- **`_contiguous` is a conservative estimate, not an exact value.** The comment is explicit: "This might be larger than the actual available space, but it will not be smaller" — so it's safe to use as an optimistic pre-check (skip a page immediately if even its `_contiguous` guess is too small) but callers must still attempt the actual allocation to get a real yes/no.
- **The mutex is externally owned and shared.** Unlike most locked classes in this codebase, `SimpleAllocator` doesn't own its `Mutex` — the constructor takes `Mutex &lock` by reference, "so the mutex can be shared where appropriate." `VertexDataPage`, for instance, is *both* a `SimpleAllocator` and a `SimpleLruPage`; sharing one lock across both roles avoids lock-ordering complexity between the two subsystems for the same object.
- **`SimpleAllocatorBlock` move-only, no copy.** The copy constructor/assignment are explicitly `= delete`d — a block represents a unique claim on a byte range, so duplicating one wouldn't make sense; move is supported for the usual container/return-by-value reasons.
- `make_block()` is `virtual` specifically so a subclass (`VertexDataPage`, `VertexDataSaveFile`) can return its own `SimpleAllocatorBlock` subclass (`VertexDataBlock`, `VertexDataSaveBlock` respectively) from `alloc()`, carrying whatever extra per-block state that subsystem needs.
- `changed_contiguous()` is a virtual hook called whenever the contiguous-space estimate changes — `VertexDataPage` overrides it to trigger `VertexDataBook::reorder_page()`, since the book's page ordering depends on this value.

## API

### `SimpleAllocator`

| Signature | Notes |
|---|---|
| `SimpleAllocator(size_t max_size, Mutex &lock)` | `lock` is caller-owned/shared, not created internally. |
| `SimpleAllocatorBlock *alloc(size_t size, size_t alignment=1)` | First-fit allocation; returns `nullptr` if no gap fits. |
| `bool is_empty() const` | No blocks currently allocated. |
| `size_t get_total_size() const` / `get_max_size() const` / `void set_max_size(size_t)` | Bytes currently allocated / capacity. |
| `size_t get_contiguous() const` | Conservative largest-free-run estimate (may overestimate, never underestimates). |
| `SimpleAllocatorBlock *get_first_block() const` | Walk allocated blocks in order. |

### `SimpleAllocatorBlock`

| Signature | Notes |
|---|---|
| `void free()` | Releases this block back to the allocator (equivalent to deleting it). |
| `SimpleAllocator *get_allocator() const` | Owning allocator. |
| `size_t get_start() const` / `get_size() const` | Byte range this block occupies. |
| `bool is_free() const` | Whether `free()` has already been called. |
| `bool realloc(size_t size)` | Attempt to resize in place (fails if the adjacent gap can't accommodate growth). |
| `SimpleAllocatorBlock *get_next_block() const` | Next allocated block, in offset order. |

## Usage

```cpp
Mutex lock;
SimpleAllocator alloc(total_capacity, lock);
SimpleAllocatorBlock *block = alloc.alloc(needed_size);
if (block != nullptr) {
  // use block->get_start()/get_size() ...
  block->free();
}
```

## See also

- [VertexDataPage](VertexDataPage.md), [VertexDataSaveFile](VertexDataSaveFile.md) — the two concrete users of this allocator
- [VertexDataBlock](VertexDataBlock.md) — `VertexDataPage`'s `SimpleAllocatorBlock` subclass
- [SimpleLru](SimpleLru.md), [AdaptiveLru](AdaptiveLru.md) — the separate eviction-policy layer, orthogonal to this pure allocation layer
