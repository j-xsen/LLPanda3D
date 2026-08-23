# BufferContext

**Source:** `panda/src/gobj/bufferContext.h` (+ `.I`, `.cxx`)
**Inherits:** [SavedContext](SavedContext.md), private LinkedListNode **Inherited by:** [IndexBufferContext](IndexBufferContext.md), [VertexBufferContext](VertexBufferContext.md), [TextureContext](TextureContext.md) (each also inherits `AdaptiveLruPage`, see [AdaptiveLru.md](AdaptiveLru.md))

Base class for the `SavedContext` subclasses that occupy an easily-measured, substantial chunk of video/AGP memory — in practice, all of them except `GeomContext`/`SamplerContext`/`ShaderContext`. `BufferContext` adds the two things every such object needs: a tracked byte size (`_data_size_bytes`, for detecting "this needs a new buffer, the size changed" and for PStats memory reporting) and a residency/active state tracked jointly with a [`BufferResidencyTracker`](BufferResidencyTracker.md) owned by the corresponding pool class (e.g. `TexturePool`'s `PreparedGraphicsObjects`).

## Behavior notes

- **Not a smart pointer to its object.** `_object` (`TypedWritableReferenceCount*`) is a raw pointer — the comment in the header explains why: the object owns its `BufferContext` (indirectly, via `PreparedGraphicsObjects`) and the `BufferContext` also references the object, so a `PT()` here would create a reference cycle.
- **State transitions move the object between chains, not just flip bits.** `set_active()`/`set_resident()` don't just set a flag — they recompute `_residency_state` (a 2-bit combination of `S_active`/`S_resident` from `BufferResidencyTracker`) and then call `set_owning_chain()` to physically unlink the object from its current `BufferContextChain` and relink it onto the chain for the new state. This is how `BufferResidencyTracker` maintains four separate linked lists (inactive/nonresident, active/nonresident, inactive/resident, active/resident) without any list traversal on state change — O(1) move via intrusive linked-list surgery (`LinkedListNode`, privately inherited).
- **`set_active(true)` implies resident.** Rendering an object is assumed to make it resident in video memory (`set_active()`'s true branch also sets the `S_resident` bit) — the two flags aren't independent in that direction; only `set_resident(false)` while still active produces the "active nonresident" ("thrashing") state that PStats specifically calls out.
- **Byte-count updates propagate to the owning chain automatically.** `update_data_size_bytes()` diffs against the previous size and calls `_owning_chain->adjust_bytes(delta)` so `BufferContextChain::get_total_size()` stays correct without a full recount.
- Construction inserts the new context onto `residency->_chains[0]` (inactive/nonresident); destruction calls `set_owning_chain(nullptr)`, which removes it from whatever chain it's currently on and decrements that chain's count/byte total.

## API

| Signature | Notes |
|---|---|
| `BufferContext(BufferResidencyTracker *residency, TypedWritableReferenceCount *object)` | Registers on `residency`'s inactive/nonresident chain. |
| `TypedWritableReferenceCount *get_object() const` | The Texture/GeomVertexArrayData/etc. this context backs. |
| `size_t get_data_size_bytes() const` | Last-reported size in bytes. |
| `void update_data_size_bytes(size_t new_size)` | Call when the backing object's size changes; adjusts the owning chain's byte total. |
| `UpdateSeq get_modified() const` / `void update_modified(UpdateSeq)` | Tracks whether the GPU-side copy is stale relative to the CPU-side object. |
| `bool get_active() const` / `void set_active(bool)` | "Rendered this frame." Setting true also implies resident. |
| `bool get_resident() const` / `void set_resident(bool)` | "Appears to be resident in video memory," per driver query results. |
| `BufferContext *get_next() const` | Walks the owning `BufferContextChain`'s intrusive list; returns `nullptr` at the chain root. |

## Usage

Application code essentially never touches `BufferContext` directly — a GSG backend calls these from within its own `prepare_texture()`/`apprehend`-style implementation:

```cpp
// inside a GSG backend's release/upload path
buffer_context->update_data_size_bytes(new_size);
buffer_context->set_active(true);   // marks resident + active this frame
```

## See also

- [BufferContextChain](BufferContextChain.md) — the intrusive linked list `BufferContext` lives on
- [BufferResidencyTracker](BufferResidencyTracker.md) — owns the four chains and their PStats reporting
- [IndexBufferContext](IndexBufferContext.md), [VertexBufferContext](VertexBufferContext.md), [TextureContext](TextureContext.md) — concrete subclasses
- [PreparedGraphicsObjects](PreparedGraphicsObjects.md) — the higher-level registry these ultimately serve
