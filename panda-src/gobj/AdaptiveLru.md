# AdaptiveLru

**Source:** `panda/src/gobj/adaptiveLru.h` (+ `.I`, `.cxx`)
**Inherits:** Namable **Inherited by:** (none directly — used as a member/composed-with type)

An LRU variant that, per its own doc comment, "attempts to avoid evicting pages that have been used more frequently (even if less recently) than other pages" — i.e. eviction priority blends *recency* with *access frequency*, rather than `SimpleLru`'s pure recency ordering. Deliberately interface-compatible with `SimpleLru` ("designed to be identical... so it may be used as a drop-in replacement"), and is in fact the LRU `PreparedGraphicsObjects::_graphics_memory_lru` uses for GPU-residency eviction (see [PreparedGraphicsObjects](PreparedGraphicsObjects.md) and the module README's [residency tracking](README.md#residency-tracking-lrus-and-allocators) section) — the higher-traffic, more performance-sensitive of the two LRU subsystems in this module.

This header actually declares **four** related classes, folded into one doc per llpanda's small-tightly-coupled-class precedent:

- **`AdaptiveLru`** — the LRU itself (documented below).
- **`AdaptiveLruPage`** — one maned page/entry (documented below).
- **`AdaptiveLruPageDynamicList`** / **`AdaptiveLruPageStaticList`** — two near-empty `LinkedListNode` subclasses that exist purely so `AdaptiveLruPage` can multiply-inherit from both and thereby sit on **two independent linked lists simultaneously** (a "sneaky C++ trick," per the header comment, since a class can't inherit `LinkedListNode` twice directly). See "Two lists, two purposes" below.

## Two lists, two purposes

Every `AdaptiveLruPage` lives on two lists at once, tracked by the owning `AdaptiveLru`:

1. **The priority-bucketed list (`_page_array[LPP_TotalPriorities]`, via `AdaptiveLruPageDynamicList`).** 50 buckets (`LruPagePriority` enum: `LPP_Highest=0` … `LPP_New=20` … `LPP_Low=40` … `LPP_TotalPriorities=50`), each a doubly-linked list of pages currently at that priority. **Lower bucket number = kept longer** (evicted last); higher = evicted sooner. Eviction (`write()`'s own comment confirms this ordering) walks buckets from `LPP_Low` toward `LPP_Highest`, i.e. least-valuable pages first.
2. **The flat incremental-update list (`_static_list`, via `AdaptiveLruPageStaticList`).** Every page, in arbitrary order, used purely so `do_partial_lru_update()` can walk a bounded number of pages per frame (`_max_updates_per_frame`, from the `adaptive-lru-max-updates-per-frame` config var) without losing its place across calls — it pops from the head and re-pushes to the tail as it processes each page, so repeated partial passes eventually cover every page in round-robin fashion instead of requiring one expensive full-list update per frame.

## Behavior notes

- **Priority starts at `LPP_New` and adapts from there.** A newly-enqueued page starts at `LPP_New` (priority 20 — "considered to be average usage of 1.0," i.e. used about once per frame). As `update_page()` processes a page (during the bounded per-frame partial update), it computes an **exponential moving average of per-frame access count** (`_average_frame_utilization`, smoothed via `_weight`, the `adaptive-lru-weight` config var) and remaps that into a new priority bucket: utilization ≥ 1.0 (used more than once/frame on average) moves toward `LPP_Highest` (harder to evict); utilization < 1.0 moves toward `LPP_Low` (easier to evict). This is the actual "adaptive" behavior the class is named for — pages that are hot even if not the *most recently* touched still get protected from eviction.
- **`enqueue_lru()`/`do_access_page()` distinguish "already accessed this frame" from "first access this frame."** Multiple accesses within the same frame just increment an in-frame usage counter (`_current_frame_usage`); the counter only rolls over to `_last_frame_usage`/resets when a *new* frame's access is detected (comparing `_current_frame_identifier` against the LRU's frame counter) — this is what lets `update_page()` later compute a genuine "accesses per frame" rate rather than "total accesses ever."
- **`begin_epoch()` does three things, matching `SimpleLru`'s interface but with more work:** runs a bounded `do_partial_lru_update()` pass (adapting priorities), evicts down to `_max_size` if currently over budget, and advances the current frame identifier from the global `ClockObject`. Unlike `SimpleLru::begin_epoch()`'s pure sentinel-marker approach, this class ties epochs directly to Panda's actual frame count.
- **Debug-build destructor deliberately skips real eviction** — it force-unlinks every remaining page from both lists without calling `evict_lru()`, specifically to avoid triggering side effects like "vertex buffers writing themselves to disk unnecessarily" during shutdown (per the `.cxx` comment) — a normal `evict_to()` call could otherwise trigger expensive last-second disk paging for data that's about to be destroyed anyway.
- Two module-local tuning constants not exposed as config vars: `HIGH_PRIORITY_SCALE = 4` and `LOW_PRIORITY_RANGE = 25`, controlling how steeply utilization above/below 1.0 maps into the priority-bucket range.

## API

### `AdaptiveLru`

| Signature | Notes |
|---|---|
| `AdaptiveLru(const std::string &name, size_t max_size)` | Weight/max-updates-per-frame default from `adaptive-lru-weight`/`adaptive-lru-max-updates-per-frame` config vars. |
| `size_t get_total_size() const` / `get_max_size() const` / `void set_max_size(size_t)` | Same shape as `SimpleLru`. |
| `size_t count_active_size() const` | Bytes among pages touched within the current epoch/frame window. |
| `void consider_evict()` / `void evict_to(size_t target_size)` | Same shape as `SimpleLru`. |
| `void begin_epoch()` | Partial priority update + over-budget eviction + frame-counter advance, in one call. |
| `void set_weight(PN_stdfloat)` / `get_weight() const` | EMA smoothing factor for utilization tracking (`AdaptiveLru`-specific, no `SimpleLru` equivalent). |
| `void set_max_updates_per_frame(int)` / `get_max_updates_per_frame() const` | Bounds the per-`begin_epoch()` priority-update work (`AdaptiveLru`-specific). |

### `AdaptiveLruPage`

| Signature | Notes |
|---|---|
| `AdaptiveLruPage(size_t lru_size)` | Same "cost" concept as `SimpleLruPage`. |
| `AdaptiveLru *get_lru() const` | Currently-owning LRU. |
| `void enqueue_lru(AdaptiveLru *lru)` / `void dequeue_lru()` | Same shape as `SimpleLruPage`. |
| `void mark_used_lru()` / `mark_used_lru(AdaptiveLru *lru)` | Same shape as `SimpleLruPage`; also feeds the per-frame usage counter driving priority adaptation. |
| `size_t get_lru_size() const` / `void set_lru_size(size_t)` | Same shape as `SimpleLruPage`. |
| `virtual void evict_lru()` | Override point, same contract as `SimpleLruPage::evict_lru()`. |
| `unsigned int get_num_frames() const` / `get_num_inactive_frames() const` | `AdaptiveLruPage`-specific: total frames since first added / frames since last accessed — not present on `SimpleLruPage`. |

## Usage

Same shape as `SimpleLru`/`SimpleLruPage` (see [SimpleLru.md](SimpleLru.md)'s usage example) — `TextureContext`/`VertexBufferContext`/`IndexBufferContext` all inherit `AdaptiveLruPage` and call `enqueue_lru()`/`mark_used_lru()` on `PreparedGraphicsObjects::_graphics_memory_lru`.

## See also

- [SimpleLru](SimpleLru.md) — the simpler, interface-compatible pure-recency alternative
- [PreparedGraphicsObjects](PreparedGraphicsObjects.md) — owns `_graphics_memory_lru`, the primary consumer of this class
- [TextureContext](TextureContext.md), [VertexBufferContext](VertexBufferContext.md), [IndexBufferContext](IndexBufferContext.md) — the `AdaptiveLruPage` subclasses tracked by that LRU
