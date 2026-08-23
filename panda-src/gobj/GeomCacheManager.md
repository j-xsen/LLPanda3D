# GeomCacheManager

**Source:** `panda/src/gobj/geomCacheManager.h` (+ `.I`, `.cxx`)

Tracks and bounds the total size of the global cache of munged Geom/vertex-
data results ([`GeomMunger`](GeomMunger.md)'s output), which would
otherwise be scattered uncounted across every `GeomVertexData` object in
the system. It's a singleton (`get_global_ptr()`) — there is exactly one
process-wide cache, sized by the `geom-cache-size` config variable.

## Behavior notes

- **The cache doesn't store the data itself** — only accounting. Cached
  results physically live on their owning objects (a `Geom`'s own
  `_cache` map, a `GeomMunger`'s format cache); `GeomCacheManager` just
  maintains one intrusive doubly-linked LRU list threading every
  [`GeomCacheEntry`](GeomCacheEntry.md) across all of those owners together,
  so eviction can be driven by one global size limit and one global
  recency order regardless of which subsystem produced the entry. This
  design deliberately lets cache data "propagate through the multiprocess
  [render] pipeline" per the header comment, rather than being centralized
  in a way that would need its own pipeline-cycling.
- **Sentinel node:** `_list` is a permanently-allocated, never-evicted
  dummy `GeomCacheEntry` whose `_prev`/`_next` serve as the head/tail of
  the ring, avoiding empty-list special-casing.
- **Eviction is two-gated:** `evict_old_entries(max_size, keep_current)`
  evicts from the LRU head (oldest) while `_total_size > max_size`, but if
  `keep_current` is true it additionally *stops early* the moment it hits
  an entry used within the last `geom-cache-min-frames` frames — i.e. an
  actively-in-use entry is protected from eviction even if the cache is
  over its size budget, at the cost of temporarily exceeding `max_size`.
  `flush()` calls this with `max_size = 0, keep_current = false` — a true
  unconditional full clear, taking `GeomMunger`'s registry lock first to
  avoid deadlock (since evicted munger-format entries may need to touch
  the munger registry).
- Destroying the singleton is explicitly disallowed
  (`nassert_raise("attempt to delete GeomCacheManager")` in the destructor).

## API

| Signature | Notes |
|---|---|
| `static GeomCacheManager *get_global_ptr()` | Singleton accessor, lazily constructs. |
| `void set_max_size(int) const` / `int get_max_size() const` | Backed by `geom-cache-size` initially. |
| `int get_total_size() const` | Current entry count. |
| `void flush()` | Unconditional full evict. |
| `void evict_old_entries()` / `evict_old_entries(max_size, keep_current)` | Trim to size, optionally protecting recently-used entries (see behavior notes). |

## See also

- [GeomCacheEntry](GeomCacheEntry.md) — the LRU node type this class manages
- [Geom](Geom.md), [GeomMunger](GeomMunger.md) — producers of cache entries
- Module README's config-var table — `geom-cache-size` / `geom-cache-min-frames`
