# PNMFileTypeRegistry

**Source:** `panda/src/pnmimage/pnmFileTypeRegistry.h` / `.cxx`
**Inherits:** *(none)*
**Inherited by:** *(none)*

A process-wide singleton that maintains the set of every known
[PNMFileType](PNMFileType.md) — one entry per supported image format (PNM,
PNG, JPEG, ...). Concrete `PNMFileType` subclasses (from the separate,
not-yet-documented `pnmimagetypes` module) register themselves here at static-
init time; [`PNMImageHeader::make_reader()`/`make_writer()`](PNMImageHeader.md)
query it to turn a filename or magic number into a usable type.

## Behavior notes

- **Access only through `get_global_ptr()`** — the constructor is `protected`,
  so there is exactly one instance, lazily created on first access.
- **Extension lookup handles `.pz`/`.gz` specially** (`get_type_from_extension()`,
  only if built `HAVE_ZLIB`): for a filename like `image.png.pz`, it strips the
  compression extension and looks up `png` instead, so a Panda-compressed
  image is still recognized by its real format.
- **Extension matching falls back to a downcased retry** — tries the extension
  as-given first, then `downcase()`'d, so `IMAGE.PNG` still resolves even
  though registered extensions are stored lowercase.
- **A bare extension (no filename) works too** — if there's no `.` in the
  string passed to `get_type_from_extension()`, the whole string is treated as
  the extension itself, so `get_type_from_extension("png")` and
  `get_type_from_extension("photo.png")` behave the same.
- **If a filename contains `/` after extension-stripping, it's treated as
  extensionless** — guards against misinterpreting a directory name with a dot
  in it as a file extension.
- **Multiple types can share one extension**; `get_type_from_extension()`
  returns the first one registered for it. `sort_preferences()` exists to let
  config-file preferences break such ties, but as of this source, it's an
  unimplemented no-op (`_requires_sort` is still tracked and would trigger a
  re-sort if it were ever filled in).
- **Duplicate registration is rejected with a warning**, not an assertion —
  `register_type()` checks `_handles` by `TypeHandle` first and returns early
  (logging via the `pnmimage` notify category) if the type is already present.
  Same for a type declaring the same extension twice.
- **`get_type_from_magic_number()` is a linear scan** over every registered
  type calling `has_magic_number()` + `matches_magic_number()`, in
  registration order — there's no magic-number index.

## API reference

| Signature | Notes |
|---|---|
| `static PNMFileTypeRegistry *get_global_ptr()` | The only way to get an instance |
| `void register_type(PNMFileType *type)` | Called by each file type's static init; warns and no-ops on duplicate |
| `void unregister_type(PNMFileType *type)` | |
| `int get_num_types() const` / `PNMFileType *get_type(int n) const` | `MAKE_SEQ`'d as `get_types()` for Python iteration |
| `PNMFileType *get_type_from_extension(const std::string &filename) const` | Accepts a bare extension or a full filename; `.pz`/`.gz`-aware |
| `PNMFileType *get_type_from_magic_number(const std::string &magic_number) const` | Linear scan, registration order |
| `PNMFileType *get_type_by_handle(TypeHandle handle) const` | Look up a previously-registered type by its C++ `TypeHandle` |
| `void write(std::ostream &out, int indent_level = 0) const` | Lists every registered type + its extensions, one per line |

## Usage

```cpp
PNMFileTypeRegistry *reg = PNMFileTypeRegistry::get_global_ptr();
PNMFileType *type = reg->get_type_from_extension("texture.png");
if (type != nullptr) {
  std::cout << "Will read/write as: " << type->get_name() << "\n";
} else {
  reg->write(std::cerr, 2);  // list what IS supported
}
```

## See also

[PNMFileType](PNMFileType.md) (what's registered) ·
[PNMImageHeader](PNMImageHeader.md) (the usual caller, via `make_reader()`/
`make_writer()`) · [README.md](README.md)
