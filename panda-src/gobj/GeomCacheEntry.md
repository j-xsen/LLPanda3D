# GeomCacheEntry

**Source:** `panda/src/gobj/geomCacheEntry.h` (+ `.I`, `.cxx`)
**Inherits:** ReferenceCount
**Inherited by:** `Geom::CacheEntry`, `GeomMunger::CacheEntry` (both
private nested classes of their respective owners — not documented
separately)

Base class for one entry in the global [`GeomCacheManager`](GeomCacheManager.md)
LRU. `GeomCacheManager` mixes several different concrete cache-entry types
(munged-geom results, munged-format results, …) together in a single LRU
list, which is why this exists as a common base rather than each cache
having its own separately-sized list — `GeomCacheManager`'s size limit
(`geom-cache-size`) applies across all of them combined.

## Behavior notes

- The entry doesn't own the cached *data* — subclasses (like `Geom::CacheEntry`)
  hold whatever payload they represent; `GeomCacheEntry` itself only
  provides the intrusive doubly-linked-list plumbing (`_prev`/`_next`) and
  `_last_frame_used` recency tracking that `GeomCacheManager` walks.
- `record()` inserts (or re-inserts) this entry at the tail of the global
  LRU list and stamps `_last_frame_used` with the current frame;
  `refresh()` is the cache-hit path — bumps `_last_frame_used` and moves
  the entry to the tail again without re-inserting. `erase()` removes it
  from the list.
- `evict_callback()` is a virtual hook called by `GeomCacheManager` right
  before an entry is dropped — the default is a no-op; subclasses use it to
  clear whatever pointer they were caching (e.g. clear a `Geom`'s cache map
  entry) since the eviction itself only manages list membership, not the
  owner's own bookkeeping.

## API

| Signature | Notes |
|---|---|
| `GeomCacheEntry()` | |
| `PT(GeomCacheEntry) record(Thread *current_thread)` | Insert/move to LRU tail, stamp recency. |
| `void refresh(Thread *current_thread)` | Cache-hit: bump recency, move to tail. |
| `PT(GeomCacheEntry) erase()` | Remove from the LRU list. |
| `virtual void evict_callback()` | Hook called just before eviction; override to clear owner-side references. |
| `virtual void output(std::ostream&) const` | |

## See also

- [GeomCacheManager](GeomCacheManager.md) — owns the global LRU list this
  class is a node in
- [Geom](Geom.md), [GeomMunger](GeomMunger.md) — the two producers of
  cache entries
