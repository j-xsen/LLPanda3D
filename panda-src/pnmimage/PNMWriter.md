# PNMWriter

**Source:** `panda/src/pnmimage/pnmWriter.h` / `.I` / `.cxx`
**Inherits:** [PNMImageHeader](PNMImageHeader.md)
**Inherited by:** concrete per-format writers (from the separate, not-yet-documented `pnmimagetypes` module)

Abstract base class for writing a specific image file format. Obtained from
[`PNMImageHeader::make_writer()`](PNMImageHeader.md), then filled in by the
caller (header fields via `set_x_size()`/`set_num_channels()`/etc. or
`copy_header_from()`) before `write_data()`/`write_row()` is called — usually
this whole dance is done for you by [`PNMImage::write()`](PNMImage.md) or
[`PfmFile::write()`](PfmFile.md).

## Behavior notes

- **Caller must fill in the header before writing** — unlike `PNMReader`
  (which gets its header from the file), a `PNMWriter` starts blank; the
  caller is responsible for calling `set_x_size()`/`set_y_size()`/
  `set_num_channels()`/`set_maxval()`/`set_color_type()` or, more simply,
  `copy_header_from(const PNMImageHeader&)` before writing any data.
  `write_data()` bails out early (returns 0) if `_x_size <= 0` or
  `_y_size <= 0`.
- **Two extension styles, same pattern as `PNMReader`**: a format either
  implements `write_data()` directly, or implements
  `supports_write_row() → true` plus `write_header()` + `write_row()`; the
  base `write_data()` default calls `write_header()` once then `write_row()`
  in a loop.
- **`set_color_type(ColorType)` is just `set_num_channels((int)type)`** — the
  enum and the channel count are the same underlying value, exactly as in
  `PNMImageHeader`.
- **Must be deleted (not just abandoned) after a successful write** — several
  format writers buffer/flush on destruction; both `write_data()`'s doc
  comment and `write_row()`'s explicitly warn that skipping the `delete` can
  lose the last unflushed data. (In Python this is automatic via reference
  counting; in raw C++ it is the caller's job.)
- **`supports_grayscale()` defaults to `true`**, but when `false`, the
  contract is that the *caller* is expected to have pre-filled the R/G/B
  fields of a grayscale `xel` array identically across all three channels
  before calling — i.e. writers that don't understand grayscale still receive
  full RGB triples, not a 1-channel buffer.
- **`supports_floating_point()`/`write_pfm()`** mirrors `PNMReader`'s
  `is_floating_point()`/`read_pfm()` — a format capable of true float output
  (PFM) implements `write_pfm(const PfmFile&)` instead of the integer path.
- **Ownership**: constructed with `owns_file`; if true, the destructor
  `delete`s the stream pointer directly (no VFS involvement on the write
  side, unlike `PNMReader`).

## API reference

| Signature | Notes |
|---|---|
| `PNMFileType *get_type() const` | |
| `void set_color_type(ColorType type)` / `void set_num_channels(int)` | Equivalent — `ColorType` is just the channel count |
| `void set_maxval(xelval)` / `void set_x_size(int)` / `void set_y_size(int)` | Must be set before writing |
| `void copy_header_from(const PNMImageHeader &header)` | Convenience: copies x/y size, channels, maxval, color space, comment, type all at once |
| `bool is_valid() const` | False if construction failed |
| `virtual bool supports_floating_point()` | Default `false` |
| `virtual bool supports_integer()` | Default `true` |
| `virtual bool write_pfm(const PfmFile &pfm)` | For floating-point formats only |
| `virtual int write_data(xel *array, xelval *alpha)` | Whole-image write; default loops `write_header()` + `write_row()` if row-capable |
| `virtual bool supports_write_row() const` | Default `false` |
| `virtual bool supports_grayscale() const` | Default `true`; if `false`, caller must pre-fill RGB from gray |
| `virtual bool write_header()` | Must precede `write_row()` calls |
| `virtual bool write_row(xel *array, xelval *alpha)` | One scanline |
| `virtual bool supports_stream_write() const` | Default `false` (requires a seekable file) |

## Usage

```cpp
// Normally obtained via PNMImageHeader::make_writer(), header populated for
// you by PNMImage::write()/PfmFile::write() — direct use looks like:
PNMWriter *writer = PNMImageHeader().make_writer("out.png");
if (writer != nullptr) {
  writer->copy_header_from(image);   // image: a PNMImageHeader-derived source
  image.write(writer);               // drives write_header()/write_row()
  delete writer;                     // required to flush!
}
```

## See also

[PNMImageHeader](PNMImageHeader.md) (base class; `make_writer()` factory) ·
[PNMImage](PNMImage.md) / [PfmFile](PfmFile.md) (typical callers of `write()`)
· [PNMReader](PNMReader.md) (read-side counterpart) ·
[PNMFileType](PNMFileType.md) (creates instances) · [README.md](README.md)
