# BamCache / BamCacheIndex / BamCacheRecord

**Source:** `panda/src/putil/bamCache.h` / `.I` / `.cxx` + `bamCacheIndex.h` / `.I` / `.cxx`
+ `bamCacheRecord.h` / `.I` / `.cxx`
**Inherits:** `BamCacheRecord : TypedWritableReferenceCount, LinkedListNode`;
`BamCacheIndex : TypedWritable, LinkedListNode`

The on-disk cache used by `ModelPool`/`TexturePool` (and anything else that
loads from a non-bam source format) to avoid re-parsing/re-converting the
same file every run. When a loader wants a file, it calls `BamCache::lookup()`
first; on a cache hit the loader gets a fully-formed `TypedWritable` back
instead of re-reading the original egg/png/etc. On a miss, the loader loads
the original file as usual, hands the result to the returned
`BamCacheRecord` via `set_data()`, and calls `BamCache::store()`.

## Behavior notes

- **`lookup()`/`store()` is a two-call protocol, not a single fetch.**
  `lookup()` always returns a non-null `BamCacheRecord` if the source file
  *could* be cached (even on a miss) — check `record->has_data()` to tell hit
  from miss. On a miss, the caller reloads the source normally, calls
  `add_dependent_file()` for every file that contributed to the result
  (including the source itself), then `record->set_data(result)` and
  `BamCache::store(record)` to persist it.
- **Staleness is determined by dependent-file timestamp+size, not content
  hashing.** `BamCacheRecord::dependents_unchanged()` walks every registered
  dependent file and compares current `(timestamp, size)` against what was
  recorded; any mismatch, or a previously-existing file now missing,
  invalidates the cache entry. `BamCache` itself doesn't call this — it's
  the caller's job to check it (typically the loader, after `lookup()`
  returns a hit) before trusting `record->get_data()`.
- **The cache filename is a hash of the absolute source path, not derived
  from its name** (`hash_filename()` — visible in `bamCache.cxx`), so cache
  entries don't collide or leak the original directory structure on disk.
  A source file already inside the cache root is never itself cached
  (`lookup()` returns `nullptr` for it).
- **Textures are cached specially.** `store()` checks whether
  `record->get_data()->is_of_type(Texture::get_class_type())`; if so it sets
  `BamWriter::BTM_rawdata` (embed actual pixels) instead of
  `BTM_fullpath` (used for everything else, which just records texture
  *references* by path) — otherwise every non-texture cache entry would
  transitively re-embed every texture it references.
- **Writes are atomic via temp-file-then-rename**: `store()` writes to
  `<cache_pathname>.<thread-unique-id>.tmp` and only then moves it into
  place, so a reader can never observe a half-written cache file — and
  concurrent writers (even across processes) never collide on the same temp
  name.
- **The index is designed for multiple processes sharing one cache
  directory, without relying on OS file locks with iostreams.** The
  index's *current filename* is itself indirected through a small
  `index_name.txt` reference file, read via
  `VirtualFileSystem::atomic_read_contents()`; `read_index_pathname()` /
  `do_read_index()` / `merge_index()` retry-and-fall-back if the pointed-to
  index file has since been replaced or deleted by another process, and
  `merge_index()` does a genuine three-way merge (by `source_pathname`)
  between the previous in-memory index and whatever's now on disk, keeping
  entries whose cache file still exists and re-reading from disk (not
  guessing) any entry that disagrees between the two.
- **`rebuild_index()` is the fallback of last resort** — a full directory
  scan of every `.bam`/`.txo` file, re-reading each one's cache-record header
  to reconstruct the index from scratch, deleting anything unreadable or
  duplicated. Triggered when the index reference can't be read at all.
- **`_index_stale_since` gates whether flushing is even attempted.**
  `mark_index_stale()` records the *first* time since the last flush the
  index changed; `consider_flush_index()` (called on every `lookup()`/
  `store()`) only actually writes if that's nonzero and `flush_time` seconds
  have elapsed — so a burst of cache activity coalesces into one disk write
  instead of one per record.
- **Cache eviction is pure LRU by access time**, tracked via
  `BamCacheRecord`'s `LinkedListNode` base (cheap O(1) move-to-front/back)
  and a `SortByAccessTime` comparator; `check_cache_size()` evicts oldest
  entries (deleting the underlying cache file) until under
  `cache_max_kbytes`, called after every `store()`/index update. A limit of
  0 kbytes means unlimited.
- **`set_read_only()` can be silently overridden.** If put into read-write
  mode but the cache directory turns out not to be writable, the cache puts
  itself back into read-only mode automatically (`emergency_read_only()`,
  invoked from `store()`'s error path) rather than repeatedly failing.
- **Global on/off is layered under four independent per-kind flags**
  (`cache_models`, `cache_textures`, `cache_compressed_textures`,
  `cache_compiled_shaders`) — each getter ANDs its own flag with the global
  `active` flag, so flipping `active` off disables caching entirely without
  losing the individual settings.
- **`BamCacheRecord`'s constructors are private** — only `BamCache` (a
  friend) creates them, via `lookup()`; application code only ever receives
  and populates records it was handed, it never constructs one directly.
  `make_copy()` explicitly drops the in-memory data pointer, since copies
  are used for index bookkeeping, not for carrying loaded objects around.

## API

### BamCache
| Signature | Notes |
|---|---|
| `BamCache()` | Reads `model-cache-*` config vars for initial settings |
| `void set_root(const Filename&)` / `Filename get_root() const` | Cache directory; created if missing |
| `bool get_active() const` / `set_active(bool)` | Global on/off |
| `get_cache_models/textures/compressed_textures/compiled_shaders() const` / matching setters | Per-kind flags, ANDed with `active` |
| `void set_cache_max_kbytes(int)` / `get_cache_max_kbytes() const` | 0 = unlimited; triggers eviction immediately when lowered |
| `void set_read_only(bool)` / `get_read_only() const` | May self-revert to `true` on write failure |
| `void set_flush_time(int)` / `get_flush_time() const` | Seconds between automatic index flushes |
| `PT(BamCacheRecord) lookup(const Filename &source_filename, const std::string &cache_extension)` | `nullptr` if uncacheable; else a record, check `has_data()` |
| `bool store(BamCacheRecord*)` | Persist a filled-in record; false on failure or read-only |
| `void consider_flush_index()` / `flush_index()` | |
| `void list_index(std::ostream&, int indent_level = 0) const` | Debug dump |
| `static BamCache *get_global_ptr()` | The instance `ModelPool`/`TexturePool` use |
| `static consider_flush_global_index()` / `flush_global_index()` | Convenience wrappers over the global instance |

### BamCacheRecord
| Signature | Notes |
|---|---|
| `const Filename &get_source_pathname() const` / `get_cache_filename() const` | |
| `time_t get_source_timestamp() const` / `get_recorded_time() const` | |
| `int get_num_dependent_files() const` / `const Filename &get_dependent_pathname(int) const` | |
| `void add_dependent_file(const Filename&)` / `(const VirtualFile*)` | Call once per file for staleness tracking |
| `void clear_dependent_files()` | |
| `bool dependents_unchanged() const` | The staleness check — caller's responsibility to call |
| `bool has_data() const` / `void clear_data()` | |
| `TypedWritable *get_data() const` | Non-owning peek |
| `bool extract_data(TypedWritable *&, ReferenceCount *&)` | Transfers ownership out |
| `void set_data(TypedWritable*, ReferenceCount*)` / `(TypedWritable*)` / `(TypedWritableReferenceCount*)` | Populate after a cache miss |
| `void output(std::ostream&) const` / `write(std::ostream&, int indent_level = 0) const` | |

### BamCacheIndex
Internal to `BamCache` (private constructor); not constructed directly by
application code. Serializes as a `TypedWritable` map of
`source_pathname → BamCacheRecord`, plus a running `_cache_size` total used
by `check_cache_size()`.

## Usage

```cpp
BamCache *cache = BamCache::get_global_ptr();
PT(BamCacheRecord) record = cache->lookup(source_filename, "bam");
if (record != nullptr) {
  if (record->has_data() && record->dependents_unchanged()) {
    TypedWritable *ptr; ReferenceCount *ref_ptr;
    record->extract_data(ptr, ref_ptr);
    // use the cached object
  } else {
    // reload the source normally...
    record->clear_dependent_files();
    record->add_dependent_file(source_filename);
    record->set_data(reloaded_object);
    cache->store(record);
  }
}
```

## See also

[BamReader.md](BamReader.md) / [BamWriter.md](BamWriter.md) (used internally
to serialize/deserialize cache files) · [TypedWritable.md](TypedWritable.md) ·
[README.md](README.md)
