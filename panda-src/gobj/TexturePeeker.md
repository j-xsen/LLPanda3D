# TexturePeeker

**Source:** `panda/src/gobj/texturePeeker.h` (+ `.I`, `.cxx`)
**Inherits:** ReferenceCount

CPU-side random-access reader into a `Texture`'s RAM image, returned by
`Texture::peek()` (private constructor — never build one directly). Use it
when app/tool code needs to read individual texel colors back (e.g. terrain
heightmap sampling, hit-testing against a texture, procedural generation
reading a source image) without going through the GPU.

## Behavior notes

- Construction picks the best available image data, in priority order: (1)
  the texture's full uncompressed RAM image if present and
  `CM_off`-compressed; (2) failing that, the low-res "simple RAM image" (see
  `simple-image-size`/`simple-image-threshold` in the
  [module config vars](README.md#config-variables-from-config_gobjh)) at
  `_component_type = T_unsigned_byte`/`F_rgba` fixed format, `_z_size = 1`;
  (3) failing both, forces a full decompress via
  `do_get_uncompressed_ram_image()`. `is_valid()` reports whether any of
  these succeeded — always check it before calling `lookup`/`fetch_pixel`,
  since an unsupported component-type/format combination leaves `_image`
  cleared and every subsequent call reads garbage/crashes.
- Because it may silently fall back to the simple image, `get_x_size()`/
  `get_y_size()` are **not guaranteed to equal the Texture's real
  dimensions** — always query them from the peeker itself rather than the
  source `Texture` when converting `(u,v)` to pixel coordinates.
- `lookup()`/`lookup_bilinear()`/`filter_rect()` take normalized `(u,v[,w])`
  in `[0,1]`; `fetch_pixel()` takes integer `(x,y[,z])` texel coordinates
  directly, unfiltered. `has_pixel()` bounds-checks integer coordinates
  before you'd call `fetch_pixel()`.
- `filter_rect()` box-filters (averages, weighted by fractional pixel
  coverage) all texels within the given normalized rectangle — used e.g. to
  downsample a region rather than pick one point sample.
- Component/texel extraction is dispatched through function pointers chosen
  once at construction time based on `_component_type` (unsigned/signed
  byte/short/int, float, half-float, packed depth-stencil) and `_format`
  (which channels exist: R, RG, RGB, RGBA, luminance(+alpha), sRGB
  variants) — this is why an unrecognized combination fails permanently at
  construction rather than per-call.
- Holds a `CPTA_uchar` (const-pointer, ref-counted array) to the image data
  it read at construction time — if the source `Texture`'s image is later
  modified or reloaded, an already-constructed `TexturePeeker` keeps
  reading its own frozen snapshot, not the new data. Call `Texture::peek()`
  again to get a fresh peeker.

## API

| Signature | Notes |
|---|---|
| `bool is_valid() const` | Check before use — false if construction couldn't find/decode readable image data. |
| `int get_x_size/get_y_size/get_z_size() const` | Dimensions of *this peeker's* backing image (may be the simple image, see notes). |
| `bool has_pixel(int x, int y[, int z]) const` | Bounds check for integer coordinates. |
| `void lookup(LColor &color, PN_stdfloat u, PN_stdfloat v[, PN_stdfloat w]) const` | Nearest-texel sample at normalized coords. |
| `void fetch_pixel(LColor &color, int x, int y[, int z]) const` | Exact-texel sample at integer coords, no filtering. |
| `bool lookup_bilinear(LColor &color, PN_stdfloat u, PN_stdfloat v) const` | Bilinear-filtered 2-D sample; returns false if out of range. |
| `void filter_rect(LColor &color, min_u, min_v[, min_w], max_u, max_v[, max_w]) const` | Box-filter average over a normalized rectangle/box. |

## Usage

```cpp
PT(TexturePeeker) peeker = tex->peek();
if (peeker != nullptr && peeker->is_valid()) {
  LColor c;
  peeker->lookup(c, 0.5f, 0.5f); // center texel
}
```

## See also

- [Texture](Texture.md)
