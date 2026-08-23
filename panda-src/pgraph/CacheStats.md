# CacheStats

**Source:** `panda/src/pgraph/cacheStats.h` / `.I` / `.cxx`

A small hit/miss/add/delete counter set used internally to track
utilization of the [RenderState](RenderState.md)/[TransformState](TransformState.md)
interning caches described in the module README. Not something application
code normally touches directly.

## Behavior notes

- **Entirely compiled out in release (`NDEBUG`) builds** — every field and
  the counting methods are `#ifndef NDEBUG`-guarded, so `CacheStats` is a
  debug/profiling-only tool with zero overhead in optimized builds.
- `init()` reads the `cache-report`/`cache-report-interval` config
  variables (deferred out of the constructor specifically to avoid relying
  on static-initialization order for the `ConfigVariable` machinery).
  `maybe_report()` (inline, in the `.I`) checks `_cache_report` and calls
  `write()` automatically once `_cache_report_interval` seconds have
  elapsed since the last `reset()`.
- `write()` divides `_total_cache_size` by `_num_states` to report an
  average cache bucket size — division by zero if no states have been
  recorded, so callers relying on this should ensure `add_num_states()` has
  been called at least once.

## API

| Method | Notes |
|---|---|
| `void init()` | Reads config vars; call once, not from a static initializer |
| `void reset(double now)` | Zeroes hit/miss/add/del counters for a new interval |
| `void write(ostream &out, const char *name) const` | Prints a one-block report |
| `void maybe_report(const char *name)` | Auto-`write()`s if the report interval has elapsed |
| `void inc_hits()` / `inc_misses()` | |
| `void inc_adds(bool is_new)` | Tracks both total adds and *new* (non-replacing) adds separately |
| `void inc_dels()` | |
| `void add_total_size(int count)` / `add_num_states(int count)` | Feed the average-size computation |

## See also

- [RenderState](RenderState.md), [TransformState](TransformState.md) — the module README's "The state pipeline" section explains what's being measured
