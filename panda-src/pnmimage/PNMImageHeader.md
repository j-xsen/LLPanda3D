# PNMImageHeader

**Source:** `panda/src/pnmimage/pnmImageHeader.h` / `.I` / `.cxx`
**Inherits:** *(none)*
**Inherited by:** [PNMImage](PNMImage.md), [PfmFile](PfmFile.md), [PNMReader](PNMReader.md), [PNMWriter](PNMWriter.md)

`PNMImageHeader` holds everything about an image *except* the pixel data
itself: size, channel count, maxval, color space, comment, and file type.
It's the sort of information you'd read from an image file's header without
decoding the body, and it's the common ancestor that lets a `PNMImage`, a
`PfmFile`, a `PNMReader`, and a `PNMWriter` all share the same size/channel/
maxval accessors and the same reader/writer-factory logic.

## Behavior notes

- **`get_color_type()` is just `_num_channels` cast to an enum.** `CT_grayscale`
  = 1 channel, `CT_two_channel` = 2 (gray+alpha), `CT_color` = 3 (RGB),
  `CT_four_channel` = 4 (RGBA). There's no independent color-type flag — a
  1-channel image *is* grayscale by definition. For a grayscale image, the
  single gray value is physically stored in the **blue** channel of the
  underlying `xel`/`pixel` struct; red and green are ignored.
- **`make_reader()`/`make_writer()` are the actual file-type-detection engine**
  used by `read_header()`/`PNMImage::read()`/`PNMImage::write()`. Detection
  order for reading: explicit `type` argument → magic number (first 2 bytes,
  looked up via [PNMFileTypeRegistry](PNMFileTypeRegistry.md)) → filename
  extension → this header's own `_type` (from a prior `set_type()`) → give up.
  For writing there's no magic-number step (nothing to sniff yet): extension →
  `_type` → give up. A freshly-made reader is checked with `is_valid()` and
  discarded (returning `nullptr`) if it fails.
- **`"-"` is a filename shorthand for stdin/stdout** in `make_reader()`/
  `make_writer()` — used by Panda's `pnmimage`-based command-line tools.
- **`compute_histogram()`/`compute_palette()` build a `PixelSpec → count` map**
  keyed on the *encoded* integer pixel value (not float), branching on
  `get_color_type()` to know which channels to read out of the raw `xel`/
  `xelval` arrays. `max_colors > 0` aborts early (returns `false`) once that
  many distinct colors have been seen — the caller (typically a quantizing
  `PNMWriter`) treats that as "too many colors to palettize." `compute_palette()`
  preserves any pixels already present in the passed-in `palette` by seeding
  the histogram with an artificially high count for them before scanning, so
  they always survive.
- **`PixelSpec` always has 4 components** (`size()` returns a constant `4`)
  regardless of the source image's channel count — unused channels are just
  zero (e.g. a grayscale-derived `PixelSpec` has `_green`/`_alpha` = 0, not
  copies of `_red`). Comparison (`operator<`, used to sort/dedupe histograms)
  is lexicographic on red, then green, then blue, then alpha.
- **`Histogram::get_pixel(n)`/`get_count(n)` are ordered most-common-first** —
  `PixelSpecCount::operator<` inverts the usual sense (`_count > other._count`)
  specifically so a `std::sort` puts the highest counts first.

## `PixelSpec` — a single pixel's integer color value

Used by `compute_histogram()`/`compute_palette()` and by
[`PNMImage::get_pixel()`/`set_pixel()`](PNMImage.md). Stores red/green/blue/
alpha as raw `xelval` (unscaled, in `[0..maxval]`), independent of any
particular image's channel count.

| Signature | Notes |
|---|---|
| `PixelSpec(xelval gray)` / `PixelSpec(xelval gray, xelval alpha)` | Grayscale constructors; alpha defaults to 0 |
| `PixelSpec(xelval r, xelval g, xelval b)` / `PixelSpec(xelval r, xelval g, xelval b, xelval a)` | |
| `PixelSpec(const xel &rgb)` / `PixelSpec(const xel &rgb, xelval alpha)` | From a raw `xel` (see [README](README.md) core concepts) |
| `bool operator<(other) const` / `==` / `!=` / `int compare_to(other) const` | Lexicographic on r, g, b, a — used for histogram sorting/dedup |
| `xelval get_red/green/blue/alpha() const` / `set_red/green/blue/alpha(xelval)` | |
| `xelval operator[](int n) const` / `static int size()` | Indexes R,G,B,A; `size()` is always 4 |

## `Histogram` — sorted color-frequency table

Populated by `PNMImage::make_histogram()` (see [PNMImage](PNMImage.md)); this
class itself is just the read-only container.

| Signature | Notes |
|---|---|
| `int get_num_pixels() const` | Number of *unique* colors, not total pixel count |
| `const PixelSpec &get_pixel(int n) const` / `int get_count(int n) const` | nth entry, ordered most-common-first |
| `int get_count(const PixelSpec &pixel) const` | O(1) lookup by color, 0 if absent |
| `void write(std::ostream &out) const` | Debug dump |

## API reference

### Type / channels
| Signature | Notes |
|---|---|
| `ColorType get_color_type() const` | `CT_invalid`, `CT_grayscale`, `CT_two_channel`, `CT_color`, `CT_four_channel` — literally `_num_channels` cast |
| `int get_num_channels() const` | 1–4 |
| `static bool is_grayscale(ColorType)` / `bool is_grayscale() const` | True for 1- or 2-channel |
| `static bool has_alpha(ColorType)` / `bool has_alpha() const` | True for 2- or 4-channel |

### Size / value range / color space
| Signature | Notes |
|---|---|
| `xelval get_maxval() const` | Max channel value (e.g. 255 or 65535); full-on |
| `ColorSpace get_color_space() const` | `CS_unspecified` by default |
| `int get_x_size/get_y_size() const` / `LVecBase2i get_size() const` | One more than the largest valid coordinate |
| `std::string get_comment() const` / `void set_comment(const std::string&)` | Free-text comment carried in some file formats |

### File type
| Signature | Notes |
|---|---|
| `bool has_type() const` / `PNMFileType *get_type() const` / `void set_type(PNMFileType*)` | Fallback type used when detection (magic number / extension) fails |

### Reading / writing (shared factory logic)
| Signature | Notes |
|---|---|
| `bool read_header(const Filename&, PNMFileType *type = nullptr, bool report_unknown_type = true)` | Reads only header info (fast — no pixel data) |
| `bool read_header(std::istream&, filename = "", type = nullptr, report_unknown_type = true)` | Stream variant; filename is advisory (extension hint) only |
| `PNMReader *make_reader(const Filename&, type = nullptr, report_unknown_type = true) const` | `"-"` means stdin. Caller owns and must `delete` the result |
| `PNMReader *make_reader(std::istream *file, owns_file = true, filename = "", magic_number = "", type = nullptr, report_unknown_type = true) const` | Lower-level: file already open |
| `PNMWriter *make_writer(const Filename&, type = nullptr) const` | `"-"` means stdout |
| `PNMWriter *make_writer(std::ostream *file, owns_file = true, filename = "", type = nullptr) const` | |
| `static bool read_magic_number(std::istream*, std::string &magic_number, int num_bytes)` | Grows `magic_number` up to `num_bytes`, preserving any already-present prefix |
| `void output(std::ostream&) const` | `"image: W by H pixels, N channels, M maxval."` |

## Usage

```cpp
// Peek at an image's dimensions without decoding pixel data.
PNMImageHeader header;
if (header.read_header("photo.png")) {
  std::cout << header.get_x_size() << "x" << header.get_y_size()
            << ", " << header.get_num_channels() << " channels\n";
}
```

## See also

[README.md](README.md) for shared pnmimage concepts (`xel`/`xelval`, color
space) · [PNMImage](PNMImage.md) (the concrete pixel-editable image) ·
[PfmFile](PfmFile.md) (the floating-point counterpart) ·
[PNMReader](PNMReader.md) / [PNMWriter](PNMWriter.md) (returned by
`make_reader()`/`make_writer()`) · [PNMFileType](PNMFileType.md) /
[PNMFileTypeRegistry](PNMFileTypeRegistry.md) (drive the type-detection logic)
