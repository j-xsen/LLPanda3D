# PNMReader

**Source:** `panda/src/pnmimage/pnmReader.h` / `.I` / `.cxx`
**Inherits:** [PNMImageHeader](PNMImageHeader.md)
**Inherited by:** concrete per-format readers (from the separate, not-yet-documented `pnmimagetypes` module)

Abstract base class for reading a specific image file format's pixel data.
Obtained from [`PNMImageHeader::make_reader()`](PNMImageHeader.md) (which has
already picked the right concrete subclass and parsed the header), then driven
through `prepare_read()` → `read_data()`/`read_row()` by
[`PNMImage::read()`](PNMImage.md) or [`PfmFile::read()`](PfmFile.md).

## Behavior notes

- **Two extension styles for subclasses**: a format either implements
  `read_data()` directly (whole image at once — the base-class default), or
  implements `supports_read_row() → true` plus `read_row()` (one scanline at a
  time from the top); the base class's own `read_data()` automatically calls
  `read_row()` in a loop when the latter is used, so most format
  implementations only need to write `read_row()`.
- **`set_read_size()` requests on-the-fly downscaling while reading**, but only
  works for row-capable readers (`supports_read_row()`), and only in
  power-of-two steps. `prepare_read()` computes the reduction as a bit-shift
  via `get_reduction_shift()` — if the requested size doesn't divide evenly
  into a power-of-two reduction of the original, no reduction is applied at
  all (silently reads full-size instead of approximating). When a reduction
  does apply, `read_data()` accumulates `2^(x_shift+y_shift)` original pixels
  per output pixel using a full-width `int` accumulator (to avoid overflow),
  then right-shifts by the combined shift to average them down — a box filter
  applied during the read itself, not a resize afterward.
  **`_x_size`/`_y_size` after `prepare_read()` reflect the reduced size**;
  `_orig_x_size`/`_orig_y_size` retain the true file dimensions and must be
  used when calling `read_row()`, which always receives full-resolution rows.
- **Grayscale/no-alpha data still needs a valid buffer** — `read_data()`
  memsets the alpha accumulator regardless, gated on `has_alpha()` (inherited
  from `PNMImageHeader`) rather than assuming the caller passed a real
  pointer only when needed.
- **`is_floating_point()`/`read_pfm()`** is the alternate path for formats
  that store true floating-point data (like PFM) — such a reader implements
  `is_floating_point() → true` and `read_pfm(PfmFile&)` instead of
  `read_data()`/`read_row()`.
- **`supports_stream_read()`** distinguishes formats that can be read from a
  general (non-seekable) stream, like a pipe, from those that need `fseek()`
  and therefore require a real file.
- **Ownership**: if constructed with `owns_file = true`, the destructor closes
  the stream via `VirtualFileSystem::close_read_file()`.

## API reference

| Signature | Notes |
|---|---|
| `void set_read_size(int x_size, int y_size)` | Suggest a downscale target; honored only by row-capable, power-of-two-friendly readers |
| `PNMFileType *get_type() const` | The type that created this reader |
| `bool is_valid() const` | False if construction/header-parse failed |
| `virtual void prepare_read()` | Call once before `read_data()`/`read_row()`; computes reduction shifts |
| `virtual bool is_floating_point()` | Default `false`; `true` ⇒ use `read_pfm()` instead |
| `virtual bool read_pfm(PfmFile &pfm)` | For floating-point formats only |
| `virtual int read_data(xel *array, xelval *alpha)` | Whole-image read; default implementation loops `read_row()` (with reduction accumulation if requested) if `supports_read_row()` |
| `virtual bool supports_read_row() const` | Default `false` |
| `virtual bool read_row(xel *array, xelval *alpha, int x_size, int y_size)` | One scanline, full original resolution; `x_size`/`y_size` here are the *original* dims, not the possibly-reduced `_x_size`/`_y_size` |
| `virtual bool supports_stream_read() const` | Default `false` (requires a seekable file) |

## Usage

```cpp
// Normally obtained via PNMImageHeader::make_reader(), not constructed
// directly — a concrete format's own factory does the real work:
PNMReader *reader = PNMImageHeader().make_reader("photo.png");
if (reader != nullptr) {
  PNMImage image;
  image.read(reader);  // PNMImage::read() drives prepare_read()/read_data()
}
```

## See also

[PNMImageHeader](PNMImageHeader.md) (base class; `make_reader()` factory) ·
[PNMImage](PNMImage.md) / [PfmFile](PfmFile.md) (typical callers of `read()`)
· [PNMWriter](PNMWriter.md) (write-side counterpart) ·
[PNMFileType](PNMFileType.md) (creates instances) · [README.md](README.md)
