# SimpleLru

**Source:** `panda/src/gobj/simpleLru.h` (+ `.I`, `.cxx`)
**Inherits:** LinkedListNode, Namable **Inherited by:** (none directly — used as a member/composed-with type; see [AdaptiveLru](AdaptiveLru.md) for the more sophisticated alternative)

A plain recency-ordered LRU: pages are evicted purely in least-recently-used order, with no notion of access frequency. Declares its companion class `SimpleLruPage` in the same header (folded into this doc rather than given a separate file, per llpanda's small-tightly-coupled-class precedent) — the "one atomic piece that may be managed by a SimpleLru chain," meant to be inherited from with `evict_lru()` overridden to do the actual eviction work. Used for `PreparedGraphicsObjects::_sampler_object_lru` (see [PreparedGraphicsObjects](PreparedGraphicsObjects.md)) and for all but the "resident" role in `VertexDataPage`'s five global LRUs (see [VertexDataPage](VertexDataPage.md)) — contrast with [AdaptiveLru](AdaptiveLru.md), the size-and-frequency-weighted alternative used for GPU residency tracking, which the class comment explicitly cross-references as offering "an identical interface... so it may be used as a drop-in replacement."

## Behavior notes

- **A global lock, not per-instance.** `SimpleLru::_global_lock` is a `static LightMutex &` — every `SimpleLru`/`SimpleLruPage` in the whole process shares one lock, unlike `AdaptiveLru` (which locks per-instance). This is a simplicity/coarse-granularity tradeoff appropriate to `SimpleLru`'s lower-traffic use cases (sampler objects, non-resident vertex-data-page bookkeeping) versus `AdaptiveLru`'s hot GPU-residency path.
- **`enqueue_lru(lru)` handles both "add" and "move to different LRU" and "just mark used."** Passing the LRU the page is already on just marks it recently-used (moves it to the list tail); passing a different (or null) LRU first removes it from wherever it currently is. This single entry point covers what would otherwise be three separate operations.
- `mark_used_lru()` (no-arg) is the common "touch this page, it was just accessed" call — equivalent to `enqueue_lru(get_lru())`. The two-arg overload additionally handles first-time or LRU-switching cases.
- **`consider_evict()` is soft; `evict_to()` is hard.** `consider_evict()` only evicts if `_total_size > _max_size` (checks before acting); `evict_to(target_size)` forces eviction down to a specific target regardless of the current max — used e.g. when temporarily lowering a budget.
- **`begin_epoch()`** resets the "active" marker (`_active_marker`, a sentinel `SimpleLruPage` used to demarcate "pages added/touched since the last epoch boundary" from older ones) — `count_active_size()` uses this to report how much of the LRU's content is from the current epoch. Analogous in spirit to `BufferResidencyTracker::begin_frame()`'s active/inactive demotion, though the mechanism (a sentinel node in the list rather than four separate chains) differs.
- `validate()` (debug builds effectively, given the `#ifndef NDEBUG`-style pattern used elsewhere in this fork) walks the whole list checking `_total_size` actually matches the sum of member pages' `_lru_size` — an internal consistency self-check, not something application code calls routinely.

## API

### `SimpleLru`

| Signature | Notes |
|---|---|
| `SimpleLru(const std::string &name, size_t max_size)` | `Namable` name is used in debug output/PStats-style reporting. |
| `size_t get_total_size() const` / `get_max_size() const` / `void set_max_size(size_t)` | Current usage / eviction budget. |
| `size_t count_active_size() const` | Bytes among pages touched since the last `begin_epoch()`. |
| `void consider_evict()` | Evict only if currently over `_max_size`. |
| `void evict_to(size_t target_size)` | Force-evict down to `target_size` regardless of current max. |
| `void begin_epoch()` | Mark the epoch boundary for `count_active_size()`. |
| `bool validate()` | Internal consistency check. |

### `SimpleLruPage`

| Signature | Notes |
|---|---|
| `SimpleLruPage(size_t lru_size)` | `lru_size` is this page's "cost" — the unit `_total_size`/`_max_size` are measured in. |
| `SimpleLru *get_lru() const` | Currently-owning LRU, or `nullptr`. |
| `void enqueue_lru(SimpleLru *lru)` | Add / move-to / mark-used-on the given LRU. |
| `void dequeue_lru()` | Remove from whatever LRU it's on. |
| `void mark_used_lru()` / `mark_used_lru(SimpleLru *lru)` | Touch (move to MRU end); optionally re-target which LRU. |
| `size_t get_lru_size() const` / `void set_lru_size(size_t)` | This page's cost/weight. |
| `virtual void evict_lru()` | Override point — actual eviction behavior, called by the LRU when this page is chosen for removal. |

## Usage

```cpp
class MyResource : public SimpleLruPage {
public:
  MyResource(size_t size) : SimpleLruPage(size) {}
  virtual void evict_lru() override {
    // free whatever this resource holds, then:
    dequeue_lru();
  }
};

SimpleLru lru("my-cache", max_bytes);
MyResource *res = new MyResource(size);
res->enqueue_lru(&lru);   // adds and marks used
lru.consider_evict();      // evicts oldest entries if over budget
```

## See also

- [AdaptiveLru](AdaptiveLru.md) — the size-and-frequency-weighted alternative with an intentionally near-identical interface
- [VertexDataPage](VertexDataPage.md) — uses `SimpleLru`/`SimpleLruPage` for its per-`RamClass` global LRUs
- [PreparedGraphicsObjects](PreparedGraphicsObjects.md) — owns `_sampler_object_lru`
