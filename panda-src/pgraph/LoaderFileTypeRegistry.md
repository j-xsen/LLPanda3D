# LoaderFileTypeRegistry

**Source:** `panda/src/pgraph/loaderFileTypeRegistry.h` / `.cxx` (no `.I`) — skips Python glue `loaderFileTypeRegistry_ext.h/.cxx`

The global registry of every known [LoaderFileType](LoaderFileType.md), keyed by
filename extension. [Loader](Loader.md) consults `get_global_ptr()` to
dispatch a load/save by extension.

## Behavior notes

- **Deferred registration:** `register_deferred_type(extension, library)`
  records an extension → shared-library-name mapping without loading
  anything. The first time `get_type_from_extension()` is asked for that
  extension and finds no real registered type, it dynamically loads the
  named library (via `load_dso()`) — presumably that library's static
  initializers then call `register_type()` for real — and retries the
  lookup. This is how optional/plugin file formats (egg, ptloader, etc.)
  stay unloaded until actually needed. `Loader::load_file_types()` seeds
  these deferred entries from the `load-file-type` config variable at
  startup (see the module README's config table).
- Extensions are matched case-insensitively (`downcase()`'d on both
  register and lookup).
- `register_type()` also scans `get_additional_extensions()` and records
  each of those extensions pointing at the same type object. Registering
  the same `LoaderFileType*` object twice is a no-op (checked by pointer
  identity), logged at debug level only.
- `write()` dumps a human-readable list of every registered type plus its
  extensions, and separately lists extensions that are only deferred
  (library not yet loaded) — this is what `Loader`'s error messages show
  when an unrecognized extension is used ("Currently known scene file
  types are:").

## API

| Method | Notes |
|---|---|
| `static LoaderFileTypeRegistry *get_global_ptr()` | |
| `void register_type(LoaderFileType *type)` | Idempotent per pointer |
| `void register_deferred_type(const string &extension, const string &library)` | Lazy-load-on-demand registration |
| `void unregister_type(LoaderFileType *type)` | |
| `int get_num_types() const` / `LoaderFileType *get_type(int n) const` | `MAKE_SEQ`'d as `get_types()` |
| `LoaderFileType *get_type_from_extension(const string &extension)` | Triggers deferred-library load if needed; null if unmatched |
| `void write(ostream &out, int indent_level = 0) const` | |

## Usage

```cpp
LoaderFileTypeRegistry *reg = LoaderFileTypeRegistry::get_global_ptr();
LoaderFileType *type = reg->get_type_from_extension("bam");
```

## See also

- [LoaderFileType](LoaderFileType.md), [LoaderFileTypeBam](LoaderFileTypeBam.md), [Loader](Loader.md)
