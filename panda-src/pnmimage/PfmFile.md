# PfmFile

**Source:** `panda/src/pnmimage/pfmFile.{h,I,cxx}`
**Inherits:** [PNMImageHeader](PNMImageHeader.md)
**Inherited by:** *(none — but `PfmVizzer`, in a different not-yet-documented module, is declared a `friend class`)*

`PfmFile` is a 2-D table of floating-point numbers — 1, 2, 3, or 4 components
per point, no maxval/gamma encoding. It's the floating-point-precision
counterpart to [PNMImage](PNMImage.md) (which stores `xelval` integers in
`[0..maxval]`), used for height maps, normal maps, depth/distance data, UV
lookup tables — anything that needs full float range/precision rather than a
displayable color. Inherits size/channel-count accessors from
[PNMImageHeader](PNMImageHeader.md); see [README.md](README.md) for shared
pnmimage concepts.

## Behavior notes

- **A "point" is the file's full per-pixel tuple** (1–4 floats, matching
  `get_num_channels()`), read/written via `get_point1/point2/point3/point4`
  (and `set_*`/`modify_*`, which return a mutable reference). This is distinct
  from `get_channel(x, y, c)`/`set_channel(...)`, which reads/writes exactly
  one component. `get_point()`/`set_point()` (no digit) is an alias for the
  3-component form. **Reading a point with a *narrower* width than the file's
  channel count is safe** (e.g. `get_point3()` on a 1-channel file returns
  `(value, 0, 0)`, `get_point4()` on a 3-channel file returns `(r,g,b,0)`) —
  the table is allocated 4 floats larger than strictly needed specifically to
  make this overread safe. **Writing a point wider than the file's channel
  count silently drops the extra components**, and writing a *narrower* point
  than the channel count only overwrites the components present in the
  passed-in point type (see `set_point2`/`set_point3`/`set_point4`'s
  per-channel-count switch in the `.I`) — the remaining channels are
  untouched, not zeroed.
- **`load(PNMImage&)`/`store(PNMImage&)` convert between float and the
  integer/gamma-encoded `PNMImage` representation** channel-for-channel
  (1→gray, 2→gray+alpha, 3→RGB via `get_xel`/`set_xel`, 4→RGBA via
  `get_xel_a`/`set_xel_a`) — i.e. through PNMImage's already-gamma-aware
  float accessors, not the raw `_val` integers. `store()` always creates the
  target `PNMImage` at `PGM_MAXMAXVAL` (65535) maxval, regardless of this
  file's own values. `read()`/`write()` on a non-PFM image type quietly route
  through a temporary `PNMImage` in the same way — you can `PfmFile::read()`
  an ordinary PNG and get a normalized-float conversion of it.
- **`store_mask(pnmimage)` writes a 1-channel 0/1 mask** from `has_point(x,y)`
  for each pixel (1 = valid, 0 = no-data). The overload taking `min_point`/
  `max_point` additionally requires every valid point's components to fall
  within that range component-wise, or it's masked to 0 too.
- **No-data handling is one of four mutually-exclusive modes**, all
  implemented by swapping a `_has_point` function pointer:
  - default (`has_point_noop`): every in-bounds point is valid.
  - `set_no_data_value(v)` (equality mode, `has_point_1..4`): a point equal to
    `v` (component-wise, exact float `==`) is "no data". `set_zero_special(true)`
    is shorthand for `set_no_data_value(LPoint4f::zero())`.
  - `set_no_data_threshold(v)` (`has_point_threshold_1..4`): a point is valid
    if **any** component is `>= v[component]` (i.e. invalid only if *all*
    components are below threshold) — note this is the opposite sense of a
    simple clamp.
  - `set_no_data_chan4(true)` (`has_point_chan4`, 4-channel only): valid iff
    channel 3 (alpha-like) is `>= 0`; a negative 4th component marks no-data.
  - `set_no_data_nan(n)` (`has_point_nan_1..4`): valid iff none of the first
    `n` components is NaN.
  `clear_no_data_value()` resets to the no-op mode. Only one mode is active at
  a time — calling any of these setters replaces whichever was active.
- **`fill_nan()`/`fill_channel_nan()` mark the whole table (or one channel) as
  no-data via NaN**, distinct from `fill_no_data_value()` which fills with
  whatever `set_no_data_value()`/`set_no_data_threshold()` value is currently
  configured (only meaningful in equality/threshold mode, not NaN mode).
- **`calc_autocrop()` finds the tight bounding box of non-empty rows/columns**
  by repeatedly testing `is_row_empty()`/`is_column_empty()` from each edge
  inward; both helpers unconditionally return `false` (nothing to crop) if no
  no-data value/threshold is configured at all — autocrop is a no-data-only
  concept, not a "trim exact zeros" operation unless zero happens to be the
  no-data value.
- **`resize()` picks a filter automatically**: `quick_filter_from()` (fast,
  fixed radius 0.5, used automatically when downscaling *and*
  `pfm_resize_quick` is true — the `config_pnmimage.h` config var), otherwise
  `gaussian_filter_from(pfm_resize_radius, ...)` if `pfm_resize_gaussian` is
  true, else `box_filter_from(pfm_resize_radius, ...)`. All three fill new
  cells created by a no-data-aware `box_filter_region`/`box_filter_line`/
  `box_filter_point` chain that weights by pixel coverage and skips
  `!has_point()` source cells entirely (so holes don't bleed into resampled
  results); if `_has_no_data_value` is set, the resized table is
  pre-filled with `_no_data_value` before filtering.
- **`xform(LMatrix4)` transforms every *valid* point as a 3-D or 4-D point**
  through the matrix (`LMatrix4::xform_point`/`xform_point_general_in_place`/
  `xform_in_place` depending on channel count) — points where `!has_point()`
  are left untouched. This is a per-point matrix transform, not a per-channel
  arithmetic op like `apply_exponent`/`operator*=`.
- **`forward_distort(dist)`/`reverse_distort(dist)` remap UV space through a
  2-component distortion map**, both by upscaling to a shared "working size"
  (`max(ceil(size*scale_factor), dist size)`) first to reduce integer-
  truncation artifacts, then resizing back down at the end. `forward_distort`
  computes `this(u,v) = this(dist(u,v))` (pulls samples via
  `calc_bilinear_point`); `reverse_distort` computes `this(u,v) = dist(this(u,v))`.
  By convention the distortion map's Y axis is inverted relative to this
  file's (`v=1` in this file ↔ `y=0` in `dist`) — used for lens/projector
  distortion correction.
- **`apply_1d_lut(channel, lut, x_scale)`** treats `lut` as an `Nx1`
  1-component file mapping `channel`'s value (scaled by `x_scale`) through a
  bilinear lookup (`lut.calc_bilinear_point(v * x_scale, 0.5)`) — a smooth
  1-D remap curve, not a hard index lookup.
- **`indirect_1d_lookup(index_image, channel, pixel_values)` is a hard (non-
  interpolated) index lookup**: `new(x,y) = pixel_values(round(index(x,y)[channel]
  * (pixel_values.x_size-1)), 0)` — nearest-index only, unlike `apply_1d_lut`'s
  bilinear sampling. Output channel count matches `pixel_values`, not `this`.
- **`merge(other)` fills only this file's holes from `other`** (same
  dimensions/channels required); it's a no-op entirely if `this` has no
  no-data value configured (nothing counts as a "hole" to fill).
  **`apply_mask(other)` is the inverse**: wherever `other` has no data, this
  file's corresponding point is set to *this* file's `_no_data_value` — a
  no-op unless *both* files have a no-data value configured.
- **`pull_spot(delta, xc, yc, xr, yr, exponent)`** adds `delta * t` to every
  point within the elliptical radius `(xr, yr)` of `(xc, yc)`, where
  `t = (1 - r)^exponent` and `r` is the normalized elliptical distance
  (`0` at center, `1` at the radius edge) — a radial falloff paint operation,
  additive rather than a hard set. `exponent` controls the falloff curve's
  sharpness (higher = tighter/sharper falloff near center). Returns the
  number of points touched.
- **`compute_planar_bounds()` fits a plane to 4 sampled corner points** (each
  itself a box-filter average via `compute_sample_point`), builds a rotation
  that puts that plane at `Y=0`, then computes either just those 4 points'
  bounding box (`points_only=true`) or every valid point's bounding box in
  the rotated frame, and returns a `BoundingHexahedron` rotated back into the
  original space — meant for frustum/projection-style bounds of a mostly-flat
  depth/position map, not a generic AABB (`calc_tight_bounds()` is the
  generic one).
- **`calc_average_point(x, y, radius)` uses nearest-neighbor hole-filling, not
  skipping**: it builds a "mini-grid" over the box, seeds it with real data
  points, then flood-fills (`fill_mini_grid`, unbounded recursion) every empty
  cell with the *nearest* known point's coordinates before averaging — so
  missing data doesn't just get excluded, it gets substituted with its
  nearest neighbor's value, then averaged in.
- **`calc_bilinear_point(x, y)` takes UV coordinates in `[0,1]`** (scaled
  internally to pixel space, offset by `-0.5` for pixel-center sampling), and
  weights only the up-to-4 surrounding samples that `has_point()` — if none of
  the 4 are valid, returns `false` with no output.

## API reference

### Construction / clear
| Signature | Notes |
|---|---|
| `PfmFile()` / `PfmFile(const PfmFile&)` / `operator=` | |
| `void clear()` | Empties to 0×0, 0 channels |
| `void clear(int x_size, int y_size, int num_channels)` | Allocates zero-filled table (+4 floats of safe overread padding); clears no-data mode |

### Read / write
| Signature | Notes |
|---|---|
| `bool read(const Filename&)` / `read(std::istream&, filename="")` / `read(PNMReader*)` | Non-PFM formats are quietly converted via a temporary `PNMImage` + `load()` |
| `bool write(const Filename&)` / `write(std::ostream&, filename="")` / `write(PNMWriter*)` | If the target writer doesn't support floating point, quietly converts via `store()` + `PNMImage` write |

### PNMImage interop
| Signature | Notes |
|---|---|
| `bool load(const PNMImage&)` | Float-converts an existing `PNMImage`'s pixels in (1→gray..4→RGBA) |
| `bool store(PNMImage&) const` | Converts to a new `PNMImage` at maxval `PGM_MAXMAXVAL` (65535) |
| `bool store_mask(PNMImage&) const` | 1-channel 0/1 validity mask from `has_point()` |
| `bool store_mask(PNMImage&, const LVecBase4f &min, const LVecBase4f &max) const` | Also requires all components within `[min,max]` |

### Point / channel accessors
| Signature | Notes |
|---|---|
| `bool has_point(x, y) const` | Dispatches through the active no-data mode |
| `PN_float32 get_channel/set_channel(x, y, c)` | Single component |
| `get_point1/2/3/4(x,y)` / `set_point1/2/3/4(...)` / `modify_point2/3/4(x,y)` | Full tuple; `get_point`/`set_point`/`modify_point` (no digit) = the 3-component form. Narrower reads on a wider file zero-pad; see Behavior notes on overread safety |

### Fill operations
| Signature | Notes |
|---|---|
| `fill(PN_float32)` / `fill(LPoint2f)` / `fill(LPoint3f)` / `fill(const LPoint4f&)` | Widens/narrows to the file's channel count |
| `fill_nan()` / `fill_channel_nan(int)` | Whole table / one channel to NaN |
| `fill_no_data_value()` | Fills with the currently-configured no-data value |
| `fill_channel(int, value)` / `fill_channel_masked(int, value)` | Masked variant only touches points where `has_point()` is already true |
| `fill_channel_masked_nan(int)` | Masked NaN fill |

### No-data value / threshold
| Signature | Notes |
|---|---|
| `set_no_data_value(LPoint4f/LPoint4d)` / `set_no_data_threshold(...)` | See Behavior notes — mutually exclusive modes |
| `set_zero_special(bool)` | Shorthand for `set_no_data_value(zero)` |
| `set_no_data_chan4(bool)` | 4-channel-only: alpha `< 0` = no data |
| `set_no_data_nan(int num_channels)` | NaN-in-first-N-channels = no data |
| `clear_no_data_value()` | Back to "every point valid" |
| `has_no_data_value()` / `has_no_data_threshold()` / `get_no_data_value() const` | |

### Analysis
| Signature | Notes |
|---|---|
| `calc_average_point(result, x, y, radius) const` | Manhattan-distance box average, holes filled by nearest-neighbor first (see Behavior notes) |
| `calc_bilinear_point(result, x, y) const` | UV-space `[0,1]` bilinear sample of up to 4 neighbors, `has_point()`-weighted |
| `calc_min_max(min, max) const` | AABB of valid points' values (not positions) |
| `calc_autocrop(x_begin, x_end, y_begin, y_end) const` / `calc_autocrop(LVecBase4f/4d &range) const` | Tight bounding rect of non-empty rows/cols; no-op ⇒ full size unless no-data mode active |
| `is_row_empty(y, x_begin, x_end) const` / `is_column_empty(x, y_begin, y_end) const` | |
| `calc_tight_bounds(min, max) const` | Generic 3-D AABB of valid points' `get_point()` values |

### Resize & filtering
| Signature | Notes |
|---|---|
| `resize(new_x, new_y)` | Auto-picks quick/gaussian/box filter — see Behavior notes |
| `box_filter_from(radius, const PfmFile&)` / `gaussian_filter_from(radius, ...)` / `quick_filter_from(const PfmFile&)` | Same trio pattern as [PNMImage](PNMImage.md)'s filters, but no-data-aware |

### Geometric transforms
| Signature | Notes |
|---|---|
| `reverse_rows()` | Flips row order in place |
| `flip(flip_x, flip_y, transpose)` | Same semantics as `PNMImage::flip` |
| `xform(LMatrix4f/LMatrix4d)` | Per-point matrix transform, skips `!has_point()` cells |
| `forward_distort(const PfmFile &dist, scale_factor=1.0)` / `reverse_distort(...)` | UV remap through a distortion map — see Behavior notes |
| `apply_1d_lut(channel, const PfmFile &lut, x_scale=1.0)` | Bilinear 1-D curve remap of one channel |

### Region operations
| Signature | Notes |
|---|---|
| `merge(const PfmFile&)` | Fills this file's holes from another (same size); no-op if no no-data mode set |
| `apply_mask(const PfmFile&)` | Punches holes in this file wherever `other` has holes |
| `copy_channel(to, const PfmFile&, from)` / `copy_channel_masked(to, other, from)` | Masked variant only copies where `other.has_point()` |
| `apply_crop(x_begin, x_end, y_begin, y_end)` | Reduces to the sub-rectangle (end-exclusive) |
| `clear_to_texcoords(x_size, y_size)` | Replaces with a 3-channel `(u+0.5, v+0.5, 0)/size` gradient |
| `copy_sub_image(copy, xto, yto, xfrom=0, yfrom=0, x_size=-1, y_size=-1)` | Only copies where source `has_point()` |
| `add_sub_image(...)` / `mult_sub_image(...)` / `divide_sub_image(...)` (+ `pixel_scale=1.0`) | Combine with existing data at destination; require both sides `has_point()`; `divide_sub_image` zeroes any NaN result |
| `int pull_spot(delta, xc, yc, xr, yr, exponent)` | Radial additive falloff — see Behavior notes; returns points-affected count |

### Bounds / sampling
| Signature | Notes |
|---|---|
| `compute_planar_bounds(center, point_dist, sample_radius, points_only) const` | Returns a `PT(BoundingHexahedron)` — see Behavior notes |
| `compute_sample_point(result, x, y, sample_radius) const` | Box-filter average around a UV point; used internally by `compute_planar_bounds` |

### Color / exponent / misc
| Signature | Notes |
|---|---|
| `gamma_correct(from, to)` / `gamma_correct_alpha(from, to)` | Shorthand for `apply_exponent(from/to)` on RGB or alpha only |
| `apply_exponent(gray)` / `apply_exponent(gray, alpha)` / `apply_exponent(c0,c1,c2)` / `apply_exponent(c0,c1,c2,c3)` | Per-channel `v = v^exponent`, no no-data skip (operates on raw table) |
| `operator*=(float)` | Per-point scalar multiply, skips `!has_point()` cells |
| `indirect_1d_lookup(index_image, channel, pixel_values)` | Hard nearest-index lookup — see Behavior notes |
| `output(std::ostream&) const` | `"floating-point image: W by H pixels, N channels."` |
| `const vector_float &get_table() const` / `void swap_table(vector_float&)` | Low-level raw access to the backing storage |

## Usage

```cpp
// Load a heightmap-style PFM, resample it, and sample a point.
PfmFile pfm;
pfm.read("heightmap.pfm");
pfm.set_no_data_value(LPoint4f(-9999, 0, 0, 0));  // sentinel "no data" value

pfm.box_filter_from(1.0, pfm);   // smooth in place (self-resample)

LPoint3f sample;
if (pfm.calc_bilinear_point(sample, 0.5f, 0.5f)) {
  // sample now holds the interpolated value at the center of the map
}

// Convert to/from a regular image.
PNMImage preview;
pfm.store(preview);
preview.write("heightmap_preview.png");
```

## See also

[PNMImageHeader](PNMImageHeader.md) (shared size/channel/type accessors) ·
[PNMImage](PNMImage.md) (integer/gamma-encoded counterpart — `load()`/`store()`
convert between the two) · [README.md](README.md) for shared pnmimage
concepts and the `pfm_*` config variables
