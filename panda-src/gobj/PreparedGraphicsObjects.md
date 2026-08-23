# PreparedGraphicsObjects

**Source:** `panda/src/gobj/preparedGraphicsObjects.h` (+ `.I`, `.cxx`, ~2.2k lines total — the biggest class in this fork)
**Inherits:** ReferenceCount **Inherited by:** (none)

Per-GSG (or per-group-of-sharing-GSGs) registry that owns every `*Context` object — see the module README's [`PreparedGraphicsObjects`/`*Context` handshake](README.md#preparedgraphicsobjects--context-handshake) for the pattern overview. This is the class application code actually calls into (indirectly, via `Texture::prepare()`/`Geom::prepare()`/etc.) to get GPU resources uploaded, and it's the central coordination point for *when* that upload/release actually happens relative to frame boundaries and thread safety. It's stored separately from `GraphicsStateGuardian` itself specifically so multiple GSGs that share a graphics context (e.g. two windows on the same GL context) can share one `PreparedGraphicsObjects` and its cached resources.

## Behavior notes

- **Seven resource types, one repeated pattern.** `Texture`, `SamplerState`, `Geom`, `Shader`, `GeomVertexArrayData` (vertex buffers), `GeomPrimitive` (index buffers), and `ShaderBuffer` each get an almost identical quintet of operations: `enqueue_X()` / `is_X_queued()` / `dequeue_X()` / `release_X()` / `release_all_Xs()`, plus `prepare_X_now()`. Once you understand the pattern for one (texture, below), the rest are the same shape with different container types. Samplers are the odd one out: since a `SamplerState` is a value type (not identity-based), prepared samplers are keyed by *value* in a `pmap<SamplerState, SamplerContext*>` rather than a pointer set.
- **Two-phase prepare: enqueue now, actually upload at `begin_frame()`.** `enqueue_texture()` just inserts into `_enqueued_textures` — no GPU work happens yet, because there's no guarantee the calling thread has the right graphics context current. `prepare_texture_now()` is the one that actually calls `gsg->prepare_texture()` and expects to be called only when the GSG's context is genuinely current (normally reached indirectly, from `begin_frame()`, not called by application code directly).
- **`begin_frame()` does releases *before* new prepares, same-frame.** Order matters: it first walks every `_released_*` set and calls the matching `gsg->release_X()`, *then* resets the four `BufferResidencyTracker`s for the new frame, *then* walks every `_enqueued_*` container and actually prepares each one (calling e.g. `tex->prepare_now(view, this, gsg)` for every view of every enqueued texture, then `gsg->update_texture(tc, true)` to force the initial upload). This means a texture released and a different texture enqueued in the same frame both get processed within one `begin_frame()` call, in that release-then-prepare order — freed GPU memory from this frame's releases is available (at the driver's discretion) to this frame's new uploads.
- **`release_X(Context*)` doesn't free anything immediately** — it just moves the context from the `_prepared_*` set into the `_released_*` set (and, for the object itself, calls e.g. `tc->get_texture()->clear_prepared(...)` and nulls `tc->_object`, since the source `Texture`/`Geom`/etc. might be destructed before the next `begin_frame()` gets around to actually releasing GPU resources). The actual `gsg->release_texture(tc)` call — which deletes the driver-side resource — only happens inside the next `begin_frame()`. This is the deferred-release-to-the-draw-thread mechanism referenced in the README.
- **Released vertex/index buffers can be cache-reused instead of freed.** `cache_unprepared_buffer()`/`get_cached_buffer()` implement an opt-in (`released-vbuffer-cache-size`/`released-ibuffer-cache-size` config vars, capped by `_support_released_buffer_cache` — off by default, turned on by the GL backend per a comment noting it's disabled for DX8/DX9) cache keyed by `(data_size_bytes, UsageHint)`: instead of destroying a released buffer's driver-side allocation, it's kept in an LRU-ordered cache (`BufferCacheLRU`) so a *different* buffer being prepared with the exact same size+hint can reuse the existing driver allocation rather than allocating fresh. This is purely a GPU-allocation-reuse optimization, unrelated to the `AdaptiveLru`/`SimpleLru` residency-eviction machinery below.
- **The `EnqueuedObject` future.** `enqueue_texture_future()`/`enqueue_shader_future()` (the other four resource types have their future variants commented out in the header — not implemented as of this codebase) return a `PT(EnqueuedObject)`, an `AsyncFuture` subclass whose result becomes available once `begin_frame()` actually prepares the object (`set_result(tc)` is called at that point). This is how `Texture::prepare()`/`Shader::prepare()`'s Python-visible-as-awaitable API is implemented under the hood. `EnqueuedObject::cancel()` dispatches to the right `dequeue_X()` based on `_object`'s `TypeHandle`, since one future class serves all resource kinds.
- **`_lock` is a `ReMutex`** (reentrant) held around essentially every method — `PreparedGraphicsObjects` is meant to be safely called from multiple threads (e.g. an app thread enqueueing a texture while the draw thread is mid-`begin_frame()`).
- **Destructor releases everything synchronously**, bypassing the normal deferred-to-`begin_frame()` path — the comment explains why: by the time a `PreparedGraphicsObjects` destructs, whatever GSG(s) owned it have likely already destructed too, so there's no context left to call driver release functions against; it just deletes the `*Context` objects directly (`delete tc` etc. — note these are raw deletes, not going through any release protocol, since there's genuinely nothing left to release against).

## API (representative — texture; other six resource types mirror this shape)

| Signature | Notes |
|---|---|
| `void enqueue_texture(Texture *tex)` | Queue for preparation at the next `begin_frame()`. |
| `bool is_texture_queued(const Texture *tex) const` / `is_texture_prepared(const Texture *tex) const` | Status checks. |
| `bool dequeue_texture(Texture *tex)` | Cancel a pending enqueue; returns false if it wasn't queued. |
| `void release_texture(TextureContext *tc)` / `void release_texture(Texture *tex)` | Move to the released set (context overload) or dispatch through the Texture (texture overload, handles either prepared-or-queued). |
| `int release_all_textures()` | Release every prepared/queued texture; returns count. |
| `TextureContext *prepare_texture_now(Texture *tex, int view, GraphicsStateGuardianBase *gsg)` | Immediate (non-deferred) prepare — normally reached via `Texture::prepare_now()`, not called directly. |
| `int get_num_queued_textures() const` / `get_num_prepared_textures() const` | Counts. |
| `PT(EnqueuedObject) enqueue_texture_future(Texture *tex)` | Enqueue + return an awaitable future for the resulting `TextureContext`. |

| Frame lifecycle | Notes |
|---|---|
| `void begin_frame(GraphicsStateGuardianBase *gsg, Thread *current_thread)` | Called by the GSG: processes all pending releases, then all pending prepares, for every resource type. |
| `void end_frame(Thread *current_thread)` | Called by the GSG: pushes residency-tracker PStats levels. |

| Memory management | Notes |
|---|---|
| `void set_graphics_memory_limit(size_t limit)` / `size_t get_graphics_memory_limit() const` | Caps `_graphics_memory_lru`'s max size; throws the `graphics_memory_limit_changed` event on change. |
| `void show_graphics_memory_lru(std::ostream&) const` | Debug dump of the `AdaptiveLru`. |
| `void show_residency_trackers(std::ostream&) const` | Debug dump of all four `BufferResidencyTracker`s. |
| `void release_all()` | Releases everything, all resource types. |
| `AdaptiveLru _graphics_memory_lru` | Public field — the size-and-recency-weighted eviction LRU shared by textures/vertex/index buffers. See [AdaptiveLru](AdaptiveLru.md). |
| `SimpleLru _sampler_object_lru` | Public field — plain recency LRU for sampler objects, capped by `sampler-object-limit`. See [SimpleLru](SimpleLru.md). |
| `BufferResidencyTracker _texture_residency`, `_vbuffer_residency`, `_ibuffer_residency`, `_sbuffer_residency` | Public fields — one per buffer-ish resource type. See [BufferResidencyTracker](BufferResidencyTracker.md). |

## Events

- `graphics_memory_limit_changed` — thrown from `set_graphics_memory_limit()` when the limit actually changes (see [event/README.md](../event/README.md) for the throw_event/EventHandler mechanism).

## Usage

Application code rarely touches this class by name — it goes through the resource classes' own `prepare()`:

```cpp
PT(PreparedGraphicsObjects) pgo = win->get_gsg()->get_prepared_objects();
my_texture->prepare(pgo);          // enqueues; actual upload happens next begin_frame()
// ... later, once uploaded ...
bool ready = my_texture->is_prepared(pgo);
```

## See also

- [SavedContext](SavedContext.md), [BufferContext](BufferContext.md), [TextureContext](TextureContext.md), [GeomContext](GeomContext.md), [SamplerContext](SamplerContext.md), [ShaderContext](ShaderContext.md), [IndexBufferContext](IndexBufferContext.md), [VertexBufferContext](VertexBufferContext.md) — the context types this class manages
- [AdaptiveLru](AdaptiveLru.md), [SimpleLru](SimpleLru.md), [BufferResidencyTracker](BufferResidencyTracker.md) — the caching/reporting subsystems this class owns instances of
- [Texture](Texture.md), [Shader](Shader.md), [Geom](Geom.md) — the objects whose `prepare()`/`prepare_now()` methods delegate here
