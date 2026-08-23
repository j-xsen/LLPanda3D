# LoaderFileTypeBam

**Source:** `panda/src/pgraph/loaderFileTypeBam.h` / `.cxx`
**Inherits:** [LoaderFileType](LoaderFileType.md)

The built-in `LoaderFileType` handler for Panda's native `.bam` format —
registered automatically so `Loader::load_sync("foo.bam")` always works
without any plugin. A thin wrapper around [BamFile](BamFile.md).

## Behavior notes

- `get_name()` → `"Bam"`, `get_extension()` → `"bam"`,
  `supports_compressed()` → `true` (bam files may be `.pz`-compressed),
  `supports_load()`/`supports_save()` both `true`.
- `load_file()`: opens the path via `BamFile::open_read()`, passes the
  caller's `LoaderOptions` through to the underlying `BamReader`
  (`set_loader_options()` — some object types read differently depending on
  these, e.g. `Character`'s `LF_convert_anim` handling), reads one node via
  `read_node()`, and if it's a [ModelRoot](ModelRoot.md), stamps it with
  the source path and file timestamp (`set_fullpath()`/`set_timestamp()`)
  — this is what lets [ModelPool](ModelPool.md)'s timestamp-based
  cache invalidation work for `.bam` files.
  If given a `BamCacheRecord`, registers the loaded path as a cache
  dependent file so cache invalidation tracks it.
- `save_file()`: opens for write, writes the single node object, and
  reports success only if both the open and the write succeeded.

## API

| Method | Notes |
|---|---|
| `LoaderFileTypeBam()` | |
| `std::string get_name() const` → `"Bam"` | |
| `std::string get_extension() const` → `"bam"` | |
| `bool supports_compressed() const` → `true` | |
| `bool supports_load() const` / `supports_save() const` → `true` / `true` | |
| `PT(PandaNode) load_file(Filename, LoaderOptions, BamCacheRecord *record) const` | |
| `bool save_file(Filename, LoaderOptions, PandaNode *node) const` | |

## See also

- [BamFile](BamFile.md), [LoaderFileType](LoaderFileType.md), [ModelRoot](ModelRoot.md)
