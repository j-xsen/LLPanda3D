# Texture

**Source:** `panda/src/gobj/texture.h` (+ `.I`, `.cxx`)
**Inherits:** `TypedWritableReferenceCount`, `Namable` (external, `putil`) **Inherited by:** [VideoTexture](VideoTexture.md)

Represents a texture object: typically a single 2-D image, but also a 1-D
strip, a 3-D volume, a 2-D array, a cube map (6 faces), a cube map array, a
1-D array, or a raw buffer texture (`TextureType`). This is the class you
touch for almost anything texture-related in Panda — loading from disk
(`read()`), building procedurally (`setup_2d_texture()` + `set_ram_image()`),
querying dimensions/format, or handing off to the GPU (`prepare()`). It is
the single largest class in the `gobj` module (~14.5k lines across
`.h`/`.I`/`.cxx`).

## Behavior notes

- **RAM image vs. GPU image, and the "reload on demand" trick.** A
  `Texture`'s pixel data can live in system RAM (`get_ram_image()`), on one
  or more GSGs' texture memory (via `prepare()`/`prepare_now()`), or both.
  The typical lifecycle: load from disk → RAM image populated → first render
  → GPU upload → RAM image freed. Freeing happens in `texture_uploaded()`
  (called by `GraphicsEngine` the frame after a successful upload), and only
  if both the global `keep-texture-ram` config var and the per-texture
  `set_keep_ram_image()` flag are false. If you then call `get_ram_image()`
  again on a texture that was loaded from a file, it transparently
  re-reads the file from disk — `has_ram_image()` tells you only whether
  that reload would be needed, not whether it would fail.
- **Lock-free-ish reload (`unlocked_ensure_ram_image()`):** when a RAM-image
  reload is needed, Panda does *not* hold the texture's own mutex during the
  (potentially slow) disk read. It makes a full `Texture` copy, releases its
  own lock, reloads into the copy, then re-acquires the lock and merges just
  the RAM-image-relevant fields back into `this` — guarded by a `_reloading`
  flag + condition variable so concurrent callers block and wait rather than
  double-reload. If, during that reload, fundamental properties changed
  (size, component count, etc. — e.g. because a `.txo` on disk changed
  shape), the merge detects the mismatch and updates the live texture
  accordingly rather than silently keeping stale metadata.
- **Every mutable operation goes through a `CData` + `PipelineCycler`.** All
  getters/setters read/write a private `CData` (pipeline-cycled data) via
  `CDReader`/`CDWriter`, not raw members — this is Panda's generic
  double(or more)-buffered-per-pipeline-stage pattern, shared with
  `RenderState`/`TransformState` in `pgraph`. `PUBLISHED` methods are thin
  wrappers; the real logic lives in the parallel `do_*` methods that take an
  explicit `CData *`.
- **`prepare()` vs. `prepare_now()`:** `prepare()` just enqueues the texture
  on the `PreparedGraphicsObjects`' async queue for upload at the start of
  the next frame (fire-and-forget, returns an `AsyncFuture`); `prepare_now()`
  synchronously creates/returns the `TextureContext` immediately, called
  internally by the GSG at draw time if the texture wasn't already prepared.
- **`~Texture()` calls `release_all()`** — every `TextureContext` this
  texture holds across every `PreparedGraphicsObjects` it was prepared into
  is released automatically on destruction; you don't need to do this
  yourself.
- **`load_related()` caches its result per suffix** in a private
  `_related_textures` map (e.g. so repeatedly asking for the `_normal`- or
  `_gloss`-suffixed sibling of a diffuse map only hits `TexturePool` once).
- **Auto-rescaling on load (`adjust_size()`):** disk-loaded images are
  passed through `texture-scale`/`texture-scale-limit`/
  `exclude-texture-scale` (global downscale, floored, with a glob-pattern
  exclusion list) and then, depending on `textures-power-2`/
  `textures-square` (or the `auto_texture_scale` argument), rounded up or
  down to a power of two / square — unless the texture was explicitly
  excluded or the request is for padding-size calculation only, in which
  case `ATS_pad` is treated as `ATS_none`. `get_tex_scale()` gives back the
  UV scale factor to compensate for any padding added this way.
- **`is_cacheable()`** is defined as "has enough info to write to the bam
  cache" — in practice, has a RAM image (or raw bam data), *not* simply
  "was successfully loaded."
- **Deprecated filter/wrap enums:** `DeprecatedFilterType`/
  `DeprecatedWrapMode` and the `FilterType`/`WrapMode` typedefs exist purely
  as aliases into [SamplerState](SamplerState.md)'s enums — `Texture` no
  longer owns filtering/wrap semantics itself, it just forwards to an
  internal `_default_sampler` (a `SamplerState`) for every
  wrap/filter/anisotropy/border-color getter/setter.

## API

Grouped by purpose; `INLINE` wrappers over `.I`/`.cxx` implementations
omitted where redundant with the summary. See the [GeomEnums shared-enum
table](README.md#shared-enums-geomenums) for `UsageHint` (used by
`setup_buffer_texture()`/`get_usage_hint()`).

### Texture's own enums

| Enum | Values (abridged) | Meaning |
|---|---|---|
| `TextureType` | `TT_1d_texture`, `TT_2d_texture`, `TT_3d_texture`, `TT_2d_texture_array`, `TT_cube_map`, `TT_buffer_texture`, `TT_cube_map_array`, `TT_1d_texture_array` | Overall shape/interpretation of the image data. |
| `ComponentType` | `T_unsigned_byte`, `T_unsigned_short`, `T_float`, `T_unsigned_int_24_8` (packed depth+stencil), `T_int`, `T_byte`, `T_short`, `T_half_float`, `T_unsigned_int` | Numeric storage of each channel. |
| `Format` | `F_rgb`/`F_rgba` (hardware-preferred bit depth) plus explicit-bit-depth variants (`F_rgb8`, `F_rgba16`, …), `F_depth_stencil`/`F_depth_component*`, `F_luminance*`, `F_srgb*`/`F_sluminance*` (gamma-corrected), integer formats (`F_r32i`, `F_rgba8i`, …), packed formats (`F_r11_g11_b10`, `F_rgb9_e5`, `F_rgb10_a2`) | Semantic meaning of channels; the bit-depth-suffixed variants are requests to the GSG for framebuffer storage, not a promise about in-memory `Texture` storage (which is always `num_components * component_width`). |
| `CompressionMode` | `CM_default`/`CM_off`/`CM_on` (generic) plus specific codecs: `CM_fxt1`, `CM_dxt1..5` (BC1-3), `CM_pvr1_2bpp`/`4bpp`, `CM_rgtc` (BC4/5), `CM_etc1`/`CM_etc2`, `CM_eac` | Requested texture-memory compression; a GSG that doesn't support the specific mode silently falls back to uncompressed. |
| `QualityLevel` | `QL_default`, `QL_fastest`, `QL_normal`, `QL_best` | Speed/quality hint, mainly consumed by the tinydisplay software renderer and RAM-image compression (`compress_ram_image()`). |

### Construction & setup

| Method | Notes |
|---|---|
| `Texture(name = "")` | Empty texture, defaults to a 2-D texture shape (call a `setup_*` to change). |
| `make_copy()` | Deep-ish copy (`make_copy_impl()` + clears `render_to_texture`, bumps modified counters); GPU copies are independent, `VideoTexture` copies animate independently. |
| `clear()` | Resets to default empty state, keeps the name. |
| `setup_texture(type, x, y, z, ComponentType, Format)` / `setup_1d_texture()` / `setup_2d_texture()` / `setup_3d_texture()` / `setup_cube_map()` / `setup_2d_texture_array()` / `setup_cube_map_array()` / `setup_buffer_texture()` | Configure shape+format before `read()`/`load()`/`set_ram_image()`. Cube map: x_size==y_size, z_size forced to 6. Cube map array: z_size forced to `num_cube_maps * 6`. |
| `generate_normalization_cube_map(size)` | Builds a cube map where each texel encodes the normalized direction to that texel — used for bump-mapping tricks. |
| `generate_alpha_scale_map()` | Builds a 1-D lookup texture for alpha-scale effects. |
| `clear_image()` / `set_clear_color()` / `get_clear_color()` / `has_clear_color()` / `clear_clear_color()` / `get_clear_data()` | Solid-color clearing without an actual image; applies lazily "the first time the texture is used" after `clear_image()`. |

### Loading & I/O

| Method | Notes |
|---|---|
| `read(fullpath, options)` / `read(fullpath, alpha_fullpath, ...)` / `read(fullpath, z, n, read_pages, read_mipmaps, ...)` | Load from disk; alpha can come from a separate grayscale file, and pages/mipmaps can be read individually or as a `#`-numbered series. |
| `write(fullpath [, z, n, write_pages, write_mipmaps])` | Inverse of `read()`; `.txo` extension writes Panda's native texture-object format instead of an image. |
| `read_txo()` / `make_from_txo()` / `write_txo()` | Panda's own serialized texture format (all pages+mipmaps in one file). |
| `read_dds()` / `read_ktx()` | Direct DirectDraw Surface / Khronos texture format readers, with `header_only` mode for metadata-only inspection. |
| `load(PNMImage/PfmFile [, z, n], options)` | Load from an already-in-memory image object rather than a file. |
| `load_sub_image(image, x, y, z, n)` | Patches a sub-region; texture must still have its RAM image and must not be compressed. |
| `store(PNMImage/PfmFile [, z, n])` | Inverse of `load()` — dumps to an in-memory image object. |
| `reload()` | Re-reads from the original disk file. |
| `load_related(suffix)` | Loads/caches a sibling texture named `<basename><suffix><ext>` (e.g. normal/gloss maps); see behavior notes. |

### Geometry & properties

| Method | Notes |
|---|---|
| `get_x_size()`/`get_y_size()`/`get_z_size()` | Texel dimensions; `y_size`/`z_size` are 1 for lower-dimensional types, `z_size` is 6 for a cube map. |
| `get_num_views()` / `set_num_views()` | Independent stacked images (e.g. stereo left/right); extra views appear as additional pages beyond `z_size`. |
| `get_num_pages()` | `z_size * num_views`. |
| `get_pad_x/y/z_size()` / `set_pad_size()` / `get_tex_scale()` | Unused-padding region added to satisfy power-of-2/square requirements; `get_tex_scale()` gives the UV compensation factor. |
| `get_orig_file_x/y/z_size()` | Original on-disk size, before any auto-rescale. |
| `get_num_components()` / `get_component_width()` | Channel count (1-4) and bytes per channel. |
| `get_format()` / `set_format()`, `get_component_type()` / `set_component_type()` | See enum table above. |

### Sampling (wrap/filter/anisotropy/border)

All of these forward to an internal `_default_sampler` ([SamplerState](SamplerState.md)); see that doc for the underlying enum semantics. `get_effective_*` variants resolve `*_default` placeholders against global config defaults.

| Method |
|---|
| `get/set_wrap_u/v/w(WrapMode)` |
| `get/set_minfilter/magfilter(FilterType)`, `get_effective_minfilter/magfilter()` |
| `get/set_anisotropic_degree()`, `get_effective_anisotropic_degree()` |
| `get/set_border_color()` |
| `get/set_default_sampler(SamplerState)` — copies the whole sampler state at once |
| `uses_mipmaps()` — true if `effective_minfilter` is one of the mipmap `FilterType`s |

### Compression & quality

| Method | Notes |
|---|---|
| `get/set_compression(CompressionMode)`, `has_compression()` | Requested GPU-memory compression; silently ignored if unsupported by the GSG. |
| `compress_ram_image()` / `uncompress_ram_image()` | CPU-side (de)compression via the `squish` library, independent of GPU compression. |
| `get/set_quality_level(QualityLevel)`, `get_effective_quality_level()` | Falls back to the `texture-quality-level` config var when `QL_default`. |
| `get/set_render_to_texture(bool)` | Marks this texture as a framebuffer render target; disallows compression. Normally set automatically by display code. |

### Mipmaps

| Method | Notes |
|---|---|
| `get_expected_num_mipmap_levels()`, `get_expected_mipmap_x/y/z_size(n)`, `get_expected_mipmap_num_pages(n)` | Computed from current size, regardless of whether mipmapping is enabled. |
| `get_num_ram_mipmap_images()`, `has_ram_mipmap_image(n)`, `get_num_loadable_ram_mipmap_images()`, `has_all_ram_mipmap_images()` | Levels actually resident in system RAM may have gaps; "loadable" means the contiguous run from level 0. |
| `get/modify/make_ram_mipmap_image(n)`, `set_ram_mipmap_image(n, ...)`, `set_ram_mipmap_pointer(n, void*, page_size)`, `clear_ram_mipmap_image(n)` / `clear_ram_mipmap_images()` | Direct per-level RAM image access; `set_ram_mipmap_pointer` binds external memory instead of copying. |
| `generate_ram_mipmap_images()` | CPU-generates the full mipmap chain from level 0 if the GPU/driver isn't asked to do it. |

### RAM image access

| Method | Notes |
|---|---|
| `has_ram_image()` / `has_uncompressed_ram_image()` / `might_have_ram_image()` | See reload-on-demand behavior note; `might_have_ram_image()` is the "best guess without side effects" check. |
| `get_ram_image()` | May trigger a transparent disk reload — see behavior notes. |
| `get_ram_image_as(format)` / `set_ram_image_as(image, format)` | Component-order conversion (e.g. "RGBA" ↔ Panda's internal BGRA-ish layout) — cannot handle compressed data. |
| `modify_ram_image()` / `make_ram_image()` | Modify-in-place vs. discard-and-reallocate; neither affects `keep_ram_image`. |
| `set_ram_image(image, compression, page_size)` | Direct replacement, optionally pre-compressed. |
| `clear_ram_image()` | Discards the RAM copy without changing format/size. |
| `get/set_keep_ram_image(bool)` | Opt out of the automatic post-upload RAM free (see behavior notes) — needed for procedurally generated textures that can't be reloaded from disk. |
| `get_ram_image_size()` / `get_ram_view_size()` / `get_ram_page_size()` / `get_expected_ram_image_size()` / `get_expected_ram_page_size()` | Actual (possibly compressed) vs. expected (uncompressed) byte sizes. |

### Simple RAM image

A small, cheap, always-available fallback 2-D preview image (e.g. for showing something before the full texture streams in).

| Method |
|---|
| `has_simple_ram_image()`, `get_simple_x/y_size()`, `get_simple_ram_image_size()`, `get_simple_ram_image()` |
| `set_simple_ram_image()`, `modify_simple_ram_image()`, `new_simple_ram_image()`, `generate_simple_ram_image()`, `clear_simple_ram_image()` |

Sizing/generation is controlled by the `simple-image-size`/`simple-image-threshold` config vars.

### GPU preparation

| Method | Notes |
|---|---|
| `prepare(PreparedGraphicsObjects*)` | Async enqueue for next-frame upload; returns an `AsyncFuture`. |
| `prepare_now(view, PreparedGraphicsObjects*, GraphicsStateGuardianBase*)` | Synchronous; returns the [TextureContext](TextureContext.md). |
| `is_prepared()` / `was_image_modified()` / `get_data_size_bytes()` / `get_active()` / `get_resident()` | Per-`PreparedGraphicsObjects` status queries, one per view. |
| `release(PreparedGraphicsObjects*)` / `release_all()` | Frees GPU-side context(s); `release_all()` is also called automatically by `~Texture()`. |
| `texture_uploaded()` | `GraphicsEngine` callback the frame after upload — frees the RAM image unless `keep-texture-ram`/`keep_ram_image` says otherwise. |

### Static helpers & misc

| Method | Notes |
|---|---|
| `up_to_power_2()` / `down_to_power_2()` | Bit-twiddling power-of-2 rounding. |
| `adjust_size(x_size&, y_size&, name, for_padding, scale)` | The full auto-rescale/pad algorithm — see behavior notes. |
| `format_texture_type()`/`string_texture_type()` and equivalents for `ComponentType`/`Format`/`CompressionMode`/`QualityLevel` | String ⟷ enum round-tripping, used by `.prc` config parsing and `write()`. |
| `is_unsigned()`, `is_specific(CompressionMode)`, `has_alpha(Format)`, `has_binary_alpha(Format)`, `is_srgb(Format)`, `is_integer(Format)` | `Format`/`ComponentType` classification predicates. |
| `estimate_texture_memory()` | Rough GPU memory footprint estimate. |
| `set/get/clear_aux_data(key)` | Arbitrary app-attached `TypedReferenceCount` payloads, not saved to bam. |
| `peek()` | Returns a [TexturePeeker](TexturePeeker.md) for random-access CPU reads into the image. |
| `is_cacheable()` | Whether there's enough data present to write to the bam cache (see behavior notes). |

## Events

None — `Texture` is not an event-throwing class. (`has_cull_callback()`/
`cull_callback()` exist for internal `CullTraverser` integration, not the
event system.)

## Usage

```cpp
// Load from disk.
PT(Texture) tex = TexturePool::load_texture("maps/brick.jpg");
tex->set_minfilter(SamplerState::FT_linear_mipmap_linear);
tex->set_wrap_u(SamplerState::WM_repeat);

// Or build one procedurally.
PT(Texture) proc = new Texture("noise");
proc->setup_2d_texture(256, 256, Texture::T_unsigned_byte, Texture::F_rgb);
proc->set_keep_ram_image(true);   // can't be reloaded from disk
PTA_uchar image = proc->modify_ram_image();
// ... fill `image` with 256*256*3 bytes ...

some_node_path.set_texture(tex);
```

## See also

- [VideoTexture](VideoTexture.md) — `Texture` subclass whose content decodes from a video stream per frame
- [TextureContext](TextureContext.md) — GSG-side handle produced by `prepare_now()`
- [PreparedGraphicsObjects](PreparedGraphicsObjects.md) — owns/tracks all `TextureContext`s for one GSG
- [TexturePool](TexturePool.md) — name-keyed cache `load_texture()` consults
- [TextureStage](TextureStage.md) — the role/blend-mode slot a `Texture` is applied through on a `RenderState`
- [TexturePeeker](TexturePeeker.md) — CPU random-access reader returned by `peek()`
- [SamplerState](SamplerState.md) — wrap/filter/anisotropy state `Texture`'s sampler methods forward to
- [README](README.md) — module overview, `GeomEnums`, config variables
