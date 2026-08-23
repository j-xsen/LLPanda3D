# LoaderFileType

**Source:** `panda/src/pgraph/loaderFileType.h` / `.cxx`
**Inherits:** `TypedObject`
**Inherited by:** [LoaderFileTypeBam](LoaderFileTypeBam.md), plus other-module handlers (egg loader, pt-loader plugins, etc. — not in `pgraph`)

Abstract base for a pluggable scene-graph file format handler. Each
supported format (bam, egg, and any format-conversion plugin) registers one
`LoaderFileType` instance with the global [LoaderFileTypeRegistry](LoaderFileTypeRegistry.md);
[Loader](Loader.md) dispatches to the type matching a file's extension.

## Behavior notes

- Default `supports_load()` is `true` but `supports_save()` is `false` — a
  handler need only override what it actually implements. The base
  `load_file()`/`save_file()` implementations just log an error and
  fail, so an unsupported operation degrades to a clean failure rather than
  a crash if called anyway.
- `_no_cache_flags`: a subclass whose *output* depends on `LoaderOptions`
  bits beyond the standard set (e.g. a format loader whose result differs
  based on `LF_convert_anim`) should OR those bits into `_no_cache_flags` in
  its constructor. `get_allow_disk_cache()`/`get_allow_ram_cache()` then
  factor those bits in alongside `LF_no_disk_cache`/`LF_no_ram_cache`,
  preventing the `Loader` from serving a cached result that doesn't
  reflect the options actually requested.
- `get_extension()` is the primary extension (e.g. `"bam"`);
  `get_additional_extensions()` (default empty) returns a
  space-separated list of further extensions the same type should also
  match, registered alongside the primary one in the registry.

## API

| Method | Notes |
|---|---|
| `virtual std::string get_name() const = 0` | Human-readable format name |
| `virtual std::string get_extension() const = 0` | Primary filename extension, no dot |
| `virtual std::string get_additional_extensions() const` | Default: none |
| `virtual bool supports_compressed() const` | Default: `false` — can it read `.pz`/`.gz` transparently |
| `virtual bool supports_load() const` / `supports_save() const` | Defaults: `true` / `false` |
| `virtual bool get_allow_disk_cache(const LoaderOptions&) const` | Factors in `_no_cache_flags` |
| `virtual bool get_allow_ram_cache(const LoaderOptions&) const` | Factors in `_no_cache_flags` |
| `virtual PT(PandaNode) load_file(Filename, LoaderOptions, BamCacheRecord *record) const` | Base: logs error, returns null |
| `virtual bool save_file(Filename, LoaderOptions, PandaNode *node) const` | Base: logs error, returns false |

## Usage

```cpp
class MyFileType : public LoaderFileType {
public:
  std::string get_name() const override { return "MyFormat"; }
  std::string get_extension() const override { return "myfmt"; }
  bool supports_load() const override { return true; }
  PT(PandaNode) load_file(const Filename &path, const LoaderOptions &options,
                          BamCacheRecord *record) const override {
    // ... parse and build a scene graph ...
  }
};

LoaderFileTypeRegistry::get_global_ptr()->register_type(new MyFileType);
```

## See also

- [LoaderFileTypeRegistry](LoaderFileTypeRegistry.md), [LoaderFileTypeBam](LoaderFileTypeBam.md), [Loader](Loader.md)
