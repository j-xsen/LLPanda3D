# ModelPool

**Source:** `panda/src/pgraph/modelPool.h` / `.I` / `.cxx`

A static-method, all-class-level cache unifying repeated loads of the same
filename to the same [ModelRoot](ModelRoot.md) pointer — analogous to
`TexturePool` for textures. `Loader` consults this automatically (unless
`LF_no_ram_cache` is set); most code doesn't call `ModelPool` directly.

## Behavior notes

- **Unlike `TexturePool`, filenames are not resolved before caching** — a
  relative path and an absolute path to the same file are treated as
  distinct cache keys. Callers wanting robust cache hits should resolve
  paths themselves or go through `Loader` (which does resolve when
  searching is enabled).
- Because a cache hit returns the **same shared node**, application code
  must copy it (`NodePath::copy_to()`) before mutating — this is exactly
  what `Loader` does automatically unless `LF_allow_instance` is passed.
  A negative cache entry (a `nullptr` `ModelRoot` for a path that failed
  to load) is stored too, so repeat failed loads of the same bad path
  don't keep re-hitting disk — `cache_check_timestamps` (config var)
  controls whether a negative or stale entry is retried against a
  currently-existing/newer file on disk.
- `load_model()` first checks the cache (`get_model()`-equivalent with
  timestamp verification), and on a miss loads synchronously via the
  global `Loader` with `LF_no_ram_cache | ~LF_search` forced (the pool
  itself becomes the RAM cache, so the `Loader`'s own RAM-cache path is
  bypassed to avoid double-bookkeeping) — then stores the result under a
  lock, re-checking for a race against another thread that loaded the same
  path concurrently before inserting.
- `garbage_collect()` sweeps out entries whose `ModelRoot` has no external
  references left (`get_model_ref_count() == 1`, i.e. only the pool itself
  holds it) or is a stored `nullptr`.
- All operations are protected by an internal `LightMutex`, so `ModelPool`
  is safe to use from the `Loader`'s background loading threads.

## API

All methods are `static`.

| Method | Notes |
|---|---|
| `bool has_model(const Filename&)` | Cache contains *any* entry (including negative) for this path |
| `bool verify_model(const Filename&)` | Cache contains a valid (non-null) entry |
| `ModelRoot *get_model(const Filename&, bool verify)` | Cache lookup only, no disk fallback |
| `ModelRoot *load_model(const Filename&, options=LoaderOptions())` | Cache-or-load-and-cache |
| `void add_model(const Filename&, ModelRoot*)` / `add_model(ModelRoot*)` | Second overload keys by the model's own fullpath |
| `void release_model(const Filename&)` / `release_model(ModelRoot*)` | Removes one entry |
| `void release_all_models()` | Clears the whole cache |
| `int garbage_collect()` | Sweeps unreferenced/negative entries, returns count removed |
| `void list_contents(ostream&)` / `list_contents()` / `void write(ostream&)` | Debug dump |

## See also

- [Loader](Loader.md), [ModelRoot](ModelRoot.md), [ModelNode](ModelNode.md)
