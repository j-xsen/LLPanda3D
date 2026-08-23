# BufferContextChain

**Source:** `panda/src/gobj/bufferContextChain.h` (+ `.I`, `.cxx`)
**Inherits:** private LinkedListNode **Inherited by:** (none)

An intrusive linked list of [`BufferContext`](BufferContext.md) objects sharing one residency/active state, plus a running byte-total and count. [`BufferResidencyTracker`](BufferResidencyTracker.md) holds exactly four of these (one per active×resident combination); `BufferContext::set_active()`/`set_resident()` move a context between two `BufferContextChain`s as its state changes. Existing purely to make PStats graphics-memory reporting fast (`O(1)` running totals instead of walking every buffer every frame).

## Behavior notes

- The chain's root node acts as both a sentinel and an accumulator — it's itself a `LinkedListNode` (constructed with `is_root=true`), and `BufferContext`s are inserted before it, forming a standard circular intrusive list. `get_first()` returns `nullptr` when `_next == this` (empty list) rather than returning the sentinel.
- `~BufferContextChain()` asserts `_total_size == 0 && _count == 0` — every chain must be fully drained (all its `BufferContext`s destroyed or moved off) before it's destroyed; this is a correctness check against leaking a dangling `_owning_chain` pointer in some `BufferContext`.
- `take_from(other)` bulk-transfers every context on `other`'s list onto `this` in one splice (`LinkedListNode::take_list_from`), fixing up each moved context's `_owning_chain` pointer as it goes, and zeroes `other`'s totals. Used by `BufferResidencyTracker::move_inactive()` each frame to bulk-demote everything that was "active" last frame into "inactive" in one O(1)-ish operation (not O(1) overall since it still touches each node to fix up `_owning_chain`, but it is a single list splice rather than repeated single-item moves).

## API

| Signature | Notes |
|---|---|
| `size_t get_total_size() const` | Running total bytes of all contexts on this chain. |
| `int get_count() const` | Running count of contexts on this chain. |
| `BufferContext *get_first()` | First context, or `nullptr` if empty; walk with `BufferContext::get_next()`. |
| `void take_from(BufferContextChain &other)` | Bulk-move every context from `other` onto `this`. |
| `void write(std::ostream &out, int indent_level) const` | Debug dump: count/size header + each context's `write()`. |

## Usage

Not constructed directly by application code — see [BufferResidencyTracker](BufferResidencyTracker.md), which owns the four instances application/GSG code actually interacts with indirectly through `BufferContext::set_active()`/`set_resident()`.

## See also

- [BufferContext](BufferContext.md) — the elements stored on this chain
- [BufferResidencyTracker](BufferResidencyTracker.md) — owns the four state-specific chains
