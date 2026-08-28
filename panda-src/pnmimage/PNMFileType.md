# PNMFileType

**Source:** `panda/src/pnmimage/pnmFileType.h` / `.cxx`
**Inherits:** `TypedWritable`
**Inherited by:** concrete per-format subclasses (PNM, PNG, JPEG, TGA, ...) — these live in the separate `panda/src/pnmimagetypes` module, not yet documented (see the root [README.md](../../README.md) status table)

`PNMFileType` is the abstract base of a family of classes, one per supported
image file format, that know how to construct a [PNMReader](PNMReader.md)/
[PNMWriter](PNMWriter.md) pair for that format. Application code almost never
touches `PNMFileType` directly — it's produced and consumed internally by
[`PNMImageHeader::make_reader()`/`make_writer()`](PNMImageHeader.md) and by
the [PNMFileTypeRegistry](PNMFileTypeRegistry.md).

## Behavior notes

- **Every method here has a harmless do-nothing default implementation.** The
  base class is usable (though useless) on its own: `get_num_extensions()`
  returns 0, `has_magic_number()`/`matches_magic_number()` return `false`,
  `make_reader()`/`make_writer()` return `nullptr`. A concrete subclass only
  needs to override what it actually supports.
- **`get_suggested_extension()` defaults to `get_extension(0)`** if the type
  declares at least one extension, otherwise an empty string — this is the
  only non-trivial default-implemented method.
- **Only `get_name()` is pure virtual** — every concrete file-type subclass
  must supply at least a name; everything else (extensions, magic number,
  actual reader/writer construction) is opt-in.
- **`init_pnm()`** is a protected hook subclasses are expected to call at the
  top of their `make_reader()`/`make_writer()` overrides, to lazily initialize
  any underlying third-party library exactly once (tracked by the static
  `_did_init_pnm` flag). In this base class it's currently a no-op.
- **Registered via `TypedWritable`'s type system**, not a runtime list — see
  [PNMFileTypeRegistry::register_type()](PNMFileTypeRegistry.md), which keys
  entries off `get_type()`'s `TypeHandle` to prevent double-registration.

## API reference

| Signature | Notes |
|---|---|
| `virtual std::string get_name() const = 0` | Pure virtual — every subclass must implement |
| `virtual int get_num_extensions() const` | Default: 0 |
| `virtual std::string get_extension(int n) const` | Without leading dot |
| `virtual std::string get_suggested_extension() const` | Default: `get_extension(0)` if any extensions exist |
| `virtual bool has_magic_number() const` | Default: `false` |
| `virtual bool matches_magic_number(const std::string&) const` | Default: `false` |
| `virtual PNMReader *make_reader(std::istream *file, bool owns_file = true, const std::string &magic_number = "")` | Default: `nullptr` (unsupported) |
| `virtual PNMWriter *make_writer(std::ostream *file, bool owns_file = true)` | Default: `nullptr` (unsupported) |

## Usage

```cpp
// Typical usage never touches PNMFileType directly — it flows through
// PNMImageHeader/PNMImage's own read()/write() overloads, which look the
// type up via the registry:
PNMFileType *type =
  PNMFileTypeRegistry::get_global_ptr()->get_type_from_extension("photo.png");
if (type != nullptr) {
  std::cout << "Detected type: " << type->get_name() << "\n";
}
```

## See also

[PNMFileTypeRegistry](PNMFileTypeRegistry.md) (holds every registered
instance) · [PNMReader](PNMReader.md) / [PNMWriter](PNMWriter.md) (created by
`make_reader()`/`make_writer()`) · [PNMImageHeader](PNMImageHeader.md) (drives
type detection) · [README.md](README.md)
