# PNMImage

**Source:** `panda/src/pnmimage/pnmImage.h` / `.I` / `.cxx` (+ `pnm-image-filter.cxx`
for the box/gaussian filter bodies)
**Inherits:** [PNMImageHeader](PNMImageHeader.md)
**Inherited by:** *(none)*

`PNMImage` is the workhorse of this module: a full in-memory, pixel-editable
2-D array of `xel`s (see [README.md](README.md) for `xel`/`xelval`), row-major
top-to-bottom/left-to-right, that can be read from and written to any
registered image file format via the inherited `read()`/`write()` machinery
(see [PNMImageHeader](PNMImageHeader.md) for the underlying reader/writer
factory logic). It is not thread-safe — use one instance from a single thread,
or protect access with a mutex.

## Behavior notes

- **Every pixel accessor comes in two flavors: `_val` (integer) and plain
  (float).** `_val` methods read/write raw `xelval`s in `[0..get_maxval()]`,
  encoded in the image's stored `ColorSpace` (linear, sRGB, or scRGB) — no
  conversion happens. The plain methods (`get_xel`/`set_xel`, `get_red`, ...)
  always work with **linearized floats in `[0..1]`**, converting through
  `to_val()`/`from_val()` (which switch on an internal `_xel_encoding` enum
  set by `setup_encoding()` whenever channels/maxval/color-space change, so
  the color-space-correct conversion — including an SSE2-accelerated uchar
  sRGB path — is picked once rather than branched on every pixel). Prefer the
  plain float accessors unless you specifically need the raw encoded value
  (e.g. bit-twiddling via `copy_channel_bits()`).
- **A grayscale image's single value lives in the blue channel** of the `xel`
  (inherited quirk from `PNMImageHeader` — see its "Behavior notes"). Both
  `get_gray_val()`/`set_gray_val()` and the generic `get_channel_val()`
  (channel order **B, G, R, A** — note: not RGBA) route through blue for
  channel 0. `get_bright()` is the color-space-correct way to get a single
  brightness value from *either* a grayscale or full-color image.
- **`set_num_channels()`/`add_alpha()`/`remove_alpha()`/`make_grayscale()`/
  `make_rgb()` all funnel through `set_color_type()`**, which is a real
  conversion, not just a reinterpretation: grayscale→color duplicates the
  gray value into R/G/B; color→grayscale computes `get_bright()` per pixel
  (`make_grayscale(rc, gc, bc)` lets you supply custom luminance weights,
  otherwise the image's own `_default_rc/gc/bc`); adding an alpha channel
  zero-fills it (fully transparent); removing one simply frees the array.
  Existing RGB/gray data always survives a channel-count change; only alpha
  is discarded or zero-initialized.
- **`premultiply_alpha()`/`unpremultiply_alpha()`** multiply/divide RGB by
  alpha per pixel and never touch the alpha channel itself.
  `unpremultiply_alpha()` explicitly skips any pixel with `alpha == 0` (left
  unchanged) to avoid a divide-by-zero.
- **`set_color_space()` is lossy going sRGB→linear** (precision loss from
  storage) and can force a new maxval — converting to `CS_scRGB` always sets
  `maxval` to 65535 regardless of the previous value. Converting between
  `CS_linear`/`CS_sRGB` at `maxval == 255` uses a fast 256-entry lookup table
  (`decode_sRGB_uchar`/`encode_sRGB_uchar` from `convert_srgb.h`); other
  maxvals fall back to a per-pixel float round-trip through `get_xel()`/
  `set_xel_val()` (i.e. through the *linear* domain either way — this method
  reinterprets `_val` data, it does not touch what `get_xel()` returns).
- **The `*_sub_image` family** (`copy_sub_image`, `blend_sub_image`,
  `add_sub_image`, `mult_sub_image`, `darken_sub_image`, `lighten_sub_image`)
  all share one internal clipping helper, `setup_sub_image()`: negative
  `x_size`/`y_size` means "rest of the source image"; a source or destination
  rectangle that runs off either image's edges is silently clipped rather
  than erroring. All but `copy_sub_image` take a `pixel_scale` that scales the
  *source* pixel's contribution before combining (e.g. `add_sub_image` with
  `pixel_scale=0.5` adds half-strength). `copy_sub_image` takes the fast path
  (direct value copy, no rescaling) only when both images share the same
  maxval *and* color space; otherwise it must go through the float domain.
- **`threshold()`** builds this image by comparing `select_image`'s given
  `channel` against `threshold` per pixel and copying from `lt` or `ge`
  accordingly. `lt`/`ge` may each be a single 1×1 pixel (broadcast as a solid
  fill), or full images the same size as `select_image`.
- **`fill_distance_inside()`/`fill_distance_outside()`** replace this image
  with a grayscale Manhattan-distance transform from `mask`'s dark/white
  pixels (per `threshold`), clamped to `radius` (which also becomes the new
  maxval — smaller radius computes faster). `fill_distance_inside` measures
  distance from the *nearest dark pixel*; with `shrink_from_border=true` the
  image border itself also counts as dark. `fill_distance_outside` measures
  distance from the *nearest white (>= threshold) pixel*. Combined with
  `threshold()`, these are how a mask gets shrunk/grown by N pixels. `mask`
  may safely alias `*this`. `do_fill_distance()` is the flood-fill worker —
  it's `PUBLISHED` but is an internal implementation detail, not meant to be
  called directly.
- **`quantize(max_colors)`** runs a median-cut color-reduction algorithm
  (`r_quantize()`) and is **only supported on images without an alpha
  channel** (asserts). It may leave *fewer* than `max_colors` unique colors,
  never more; a no-op if the image already has ≤ `max_colors` colors.
- **`perlin_noise_fill()`** has two overloads: `(sx, sy, table_size, seed)`
  builds a one-shot `PerlinNoise2` internally (sx/sy are in multiples of the
  image size); the `StackedPerlinNoise2&` overload reuses/shares a
  caller-supplied noise generator (e.g. to layer octaves or reuse across
  multiple fills). Both write into the grayscale/RGB channels via `set_xel()`
  with noise remapped from `[-1,1]` to `[0,1]`.
- **`remix_channels(const LMatrix4 &conv)`** requires a 3- or 4-channel
  (color) image; each pixel's `(r,g,b)` is passed through
  `conv.xform_point()` — i.e. treated as a homogeneous point, so `conv` can
  both linearly mix channels and add a constant offset.
- **The arithmetic operators (`+`, `-`, `*`, `+=`, `-=`, `*=`)** work per
  pixel; the `PNMImage`-vs-`PNMImage` forms require matching size and operate
  fastest (raw integer add/sub) when both maxval and color space are
  `CS_linear`-equal, otherwise falling back to the float domain. Multiplying
  by another `PNMImage` or an `LColorf` multiplies in the **`[0..1]`
  normalized range**, not raw encoded values. `operator~()` (bitwise
  complement, `_maxval - value` per channel including alpha) is explicitly
  documented as **not color-space correct** — it inverts the raw encoded
  value, not the linear one.
- **`read()`/`set_read_size()`**: if a requested read size can't be honored
  directly by the file-type reader (e.g. only JPEG supports scale-on-decode),
  `read()` loads at full size and then applies `quick_filter_from()` to
  resize afterward — same visible result, more memory/CPU either way. If the
  source turns out to be a floating-point format (PFM), `read()` transparently
  routes through a temporary [PfmFile](PfmFile.md) and calls `pfm.store()`
  into this image; likewise `write()` routes through `PfmFile::load()` when
  the target writer only supports floating point.
- **Grayscale images written to a format without native grayscale support**
  (`!writer->supports_grayscale()`) are silently expanded to 3 channels
  in-place (gray value copied to R/G/B) right before `write_data()` — this
  mutates the `PNMImage` object even though `write()` is logically `const`
  (achieved via a `const_cast`-style raw pointer cast internally).

## Row / CRow — indexed pixel-row proxies

`operator[](int y)` returns a lightweight `Row` (non-const `*this`) or `CRow`
(const `*this`) that wraps a reference to the image and a row index, so
`image[y][x]` reads/writes pixel `(x, y)` like a 2-D array. `Row`/`CRow`
themselves hold no pixel data — they're thin accessor objects constructed on
every subscript.

| Signature | Notes |
|---|---|
| `size_t size() const` | Row width, i.e. `image.get_x_size()` |
| `LColorf operator[](int x) const` | RGBA float `[0..1]`, equivalent to `get_xel_a(x, y)` |
| `void __setitem__(int x, const LColorf&)` *(Python only, `Row`)* | Equivalent to `set_xel_a(x, y, v)`; alpha ignored if the image has none |
| `xel &get_xel_val(int x)` *(`Row`)* / `xel get_xel_val(int x) const` *(`CRow`)* | Raw encoded pixel; `Row`'s returns a reference |
| `void set_xel_val(int x, const xel&)` *(`Row` only)* | |
| `xelval get_alpha_val(int x) const` / `void set_alpha_val(int x, xelval)` *(`Row` only)* | |

## API reference

### Construction
| Signature | Notes |
|---|---|
| `PNMImage()` | Empty/invalid until `clear(...)` or `read()` |
| `explicit PNMImage(const Filename&, PNMFileType *type = nullptr)` | Immediately reads; logs an error (doesn't throw) if the read fails |
| `explicit PNMImage(int x_size, int y_size, int num_channels = 3, xelval maxval = 255, PNMFileType *type = nullptr, ColorSpace color_space = CS_linear)` | Allocates a black (zero-filled) image |
| `PNMImage(const PNMImage &copy)` / `operator=` | Deep copy via `copy_from()` |

### Value conversion helpers
| Signature | Notes |
|---|---|
| `xelval clamp_val(int) const` | Clamps to `[0, get_maxval()]` |
| `xel to_val(const LRGBColorf&) const` / `xelval to_val(float) const` | `[0..1]` linear → encoded, color-space aware |
| `xelval to_alpha_val(float) const` | Alpha is always linear regardless of image color space |
| `LRGBColorf from_val(const xel&) const` / `float from_val(xelval) const` | Encoded → `[0..1]` linear |
| `float from_alpha_val(xelval) const` | |

### Clear / copy / fill
| Signature | Notes |
|---|---|
| `void clear()` | Frees pixel data; image becomes invalid (`is_valid() == false`) |
| `void clear(int x_size, int y_size, num_channels=3, maxval=255, type=nullptr, color_space=CS_linear)` | Re-inits to a black image of the given shape |
| `void copy_from(const PNMImage&)` | Deep copy (header + pixels) |
| `void copy_channel(const PNMImage&, int src_channel, int dest_channel)` | Per-pixel single-channel copy; images must match in size |
| `void copy_channel_bits(const PNMImage&, src_channel, dest_channel, xelval src_mask, int right_shift)` | Bitwise-masked channel copy on raw encoded values; negative `right_shift` means left-shift |
| `void copy_header_from(const PNMImageHeader&)` | Reallocates pixel storage **uninitialized** — header only, no pixel data |
| `void take_from(PNMImage &orig)` | Steals `orig`'s arrays (no copy); empties `orig` |
| `void fill(r, g, b)` / `void fill(gray = 0.0)` | Float `[0..1]`, doesn't touch alpha |
| `void fill_val(xelval r, g, b)` / `void fill_val(xelval gray = 0)` | Raw encoded |
| `void alpha_fill(float alpha = 0.0)` / `void alpha_fill_val(xelval alpha = 0)` | Adds an alpha channel first if the image doesn't have one |

### Read-size / color space
| Signature | Notes |
|---|---|
| `void set_read_size(int x, int y)` / `clear_read_size()` / `bool has_read_size() const` | Requests decode-time downscale on the *next* `read()` (see Behavior notes) |
| `int get_read_x_size/get_read_y_size() const` | Requested size if set, else actual image size |
| `ColorSpace get_color_space() const` | |

### File I/O
| Signature | Notes |
|---|---|
| `bool read(const Filename&, PNMFileType *type = nullptr, bool report_unknown_type = true)` | |
| `bool read(std::istream&, filename = "", type = nullptr, report_unknown_type = true)` | |
| `bool read(PNMReader *reader)` | Reader is always deleted, success or failure |
| `bool write(const Filename&, PNMFileType *type = nullptr) const` | |
| `bool write(std::ostream&, filename = "", type = nullptr) const` | |
| `bool write(PNMWriter *writer) const` | Writer is always deleted |

### Validity / channel & color-type management
| Signature | Notes |
|---|---|
| `bool is_valid() const` | True once pixel storage is allocated |
| `void set_num_channels(int)` / `void set_color_type(ColorType)` | See Behavior notes — real data conversion, not reinterpretation |
| `void set_color_space(ColorSpace)` | May be lossy; may force a new maxval (scRGB → 65535) |
| `void add_alpha()` / `void remove_alpha()` | |
| `void make_grayscale()` / `make_grayscale(rc, gc, bc)` / `void make_rgb()` | |
| `void premultiply_alpha()` / `void unpremultiply_alpha()` | |
| `void reverse_rows()` | Flips vertically (Y axis) |
| `void flip(bool flip_x, bool flip_y, bool transpose)` | `transpose` swaps X/Y (image dimensions swap too); combine for any 90°/180° rotation |
| `void set_maxval(xelval)` | Rescales all encoded values (and alpha) to the new range |

### Pixel access — encoded (`_val`)
| Signature | Notes |
|---|---|
| `xel &get_xel_val(x, y)` / `xel get_xel_val(x, y) const` | Non-const overload returns a mutable reference |
| `void set_xel_val(x, y, const xel&)` / `set_xel_val(x, y, r, g, b)` / `set_xel_val(x, y, gray)` | |
| `xelval get_red/green/blue/gray_val(x, y) const` | `get_gray_val` reads blue channel (see Behavior notes) |
| `xelval get_alpha_val(x, y) const` | Error to call unless `has_alpha()` |
| `void set_red/green/blue/gray/alpha_val(x, y, xelval)` | |
| `xelval get_channel_val(x, y, channel) const` / `void set_channel_val(...)` | Channel order **B, G, R, A** |
| `float get_channel(x, y, channel) const` / `void set_channel(...)` | Same channel order, float `[0..1]` |
| `PixelSpec get_pixel(x, y) const` / `void set_pixel(x, y, const PixelSpec&)` | See [PNMImageHeader](PNMImageHeader.md) `PixelSpec` |

### Pixel access — linear float
| Signature | Notes |
|---|---|
| `LRGBColorf get_xel(x, y) const` / `set_xel(x, y, ...)` (3 overloads) | |
| `LColorf get_xel_a(x, y) const` / `set_xel_a(x, y, ...)` (2 overloads) | RGBA together |
| `float get_red/green/blue/gray/alpha(x, y) const` | |
| `void set_red/green/blue/gray/alpha(x, y, float)` | |
| `float get_bright(x, y) const` | Uses the image's own default luminance weights; correct for both grayscale and color |
| `float get_bright(x, y, rc, gc, bc) const` | Color images only |
| `float get_bright(x, y, rc, gc, bc, ac) const` | Four-channel images only |

### Blending
| Signature | Notes |
|---|---|
| `void blend(x, y, const LRGBColorf&, float alpha)` / `blend(x, y, r, g, b, float alpha)` | Standard over-compositing; `alpha=1` replaces outright, `alpha=0` no-ops |

### Sub-image operations
| Signature | Notes |
|---|---|
| `void copy_sub_image(copy, xto, yto, xfrom=0, yfrom=0, x_size=-1, y_size=-1)` | |
| `void blend_sub_image(copy, xto, yto, xfrom=0, yfrom=0, x_size=-1, y_size=-1, pixel_scale=1.0)` | |
| `void add_sub_image(...)` / `mult_sub_image(...)` / `darken_sub_image(...)` / `lighten_sub_image(...)` | Same signature shape as `blend_sub_image`; darken/lighten take per-channel min/max against existing pixels |
| `void threshold(select_image, channel, threshold, lt, ge)` | See Behavior notes |
| `void fill_distance_inside(mask, threshold, radius, shrink_from_border)` / `fill_distance_outside(mask, threshold, radius)` | See Behavior notes |
| `void indirect_1d_lookup(index_image, channel, pixel_values)` | `pixel_values` is typically an Nx1 LUT image; nearest-index only, no interpolation |
| `void rescale(float min_val, float max_val)` | Linearly remaps RGB so `[min_val,max_val] → [0,1]`, clamped; alpha untouched |
| `void copy_channel(copy, xto, yto, cto, xfrom=0, yfrom=0, cfrom=0, x_size=-1, y_size=-1)` | Sub-image + single-channel overload of the simpler `copy_channel` above |
| `void render_spot(fg, bg, min_radius, max_radius)` | Draws an antialiased radial gradient spot centered in the image, in normalized `[-1,1]`-ish radius units |
| `void expand_border(left, right, bottom, top, const LColorf&)` | Negative values crop instead of expand |

### Filtering / resampling
| Signature | Notes |
|---|---|
| `void box_filter(float radius = 1.0)` / `void gaussian_filter(float radius = 1.0)` | In-place blur (filters over the whole image without resizing) |
| `void unfiltered_stretch_from(const PNMImage&)` | Nearest-point resize (no filtering) into the current size |
| `void box_filter_from(radius, const PNMImage&)` / `void gaussian_filter_from(radius, const PNMImage&)` | Filtered resize from another image into this one's current size |
| `void quick_filter_from(const PNMImage&, xborder=0, yborder=0)` | Fast box-ish resize; used internally by `read()` when a reader can't honor `set_read_size()` directly |

### Histogram / quantize / noise / color transforms
| Signature | Notes |
|---|---|
| `void make_histogram(Histogram&)` | See [PNMImageHeader::Histogram](PNMImageHeader.md) |
| `void quantize(size_t max_colors)` | Median-cut; alpha-less images only (see Behavior notes) |
| `void perlin_noise_fill(sx, sy, table_size=256, seed=0)` | |
| `void perlin_noise_fill(StackedPerlinNoise2&)` | |
| `void remix_channels(const LMatrix4&)` | 3/4-channel images only |
| `void gamma_correct(from_gamma, to_gamma)` / `gamma_correct_alpha(from_gamma, to_gamma)` | Convenience wrappers around `apply_exponent(from/to)` |
| `void apply_exponent(gray_exp)` / `(gray_exp, alpha_exp)` / `(r_exp, g_exp, b_exp)` / `(r_exp, g_exp, b_exp, alpha_exp)` | `L' = L ^ exponent`, per channel |

### Averages
| Signature | Notes |
|---|---|
| `LRGBColorf get_average_xel() const` | Mean of `get_xel()` over all pixels |
| `LColorf get_average_xel_a() const` | Mean including alpha |
| `float get_average_gray() const` | Mean of `get_gray()` |

### Operators
| Signature | Notes |
|---|---|
| `PNMImage operator~() const` | Bitwise complement of encoded values; **not** color-space correct |
| `operator + / - / *` with `PNMImage` or `LColorf`, `* float` | Return a new image; implemented via the `+=`/`-=`/`*=` in-place forms |
| `operator += / -= / *=` with `PNMImage` or `LColorf` | In-place; `PNMImage` operands must match size; `*` in the normalized `[0..1]` domain |

## Usage

```cpp
PNMImage img;
if (img.read("photo.png")) {
  // Brighten by scaling toward white, then blur slightly.
  img *= LColorf(1.2f, 1.2f, 1.2f, 1.0f);
  img.gaussian_filter(1.5f);

  // Poke a single pixel directly.
  img.set_xel(0, 0, 1.0f, 0.0f, 0.0f);

  img.write("photo_edited.png");
}
```

## See also

[PNMImageHeader](PNMImageHeader.md) (inherited size/type/`PixelSpec`/
`Histogram`) · [PfmFile](PfmFile.md) (floating-point counterpart; `read()`/
`write()` transparently interop with it for PFM files) · [PNMPainter](PNMPainter.md)
(draws shapes into a `PNMImage`) · [PNMReader](PNMReader.md) / [PNMWriter](PNMWriter.md)
(returned by the inherited `make_reader()`/`make_writer()`) · [README.md](README.md)
