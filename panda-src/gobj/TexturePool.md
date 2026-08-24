# TexturePool

**Source:** `panda/src/gobj/texturePool.h` (+ `.I`, `.cxx`)
**Inherits:** (none — static singleton, private constructor)

The preferred entry point for loading textures from image files: a single
global pool (`get_global_ptr()`) that unifies references to the same
filename so multiple models sharing a texture don't waste GPU/RAM loading
and storing it twice. This is the class app/loader code actually calls —
`Texture::make_texture()`/individual format readers are lower-level.

## Behavior notes

- **`get_texture()` vs. `load_texture()`:** `get_texture()` only returns an
  already-loaded `Texture*` or `nullptr` — it never touches disk.
  `load_texture()` (marked `BLOCKING`) loads from disk if not already
  cached, returning the shared existing `Texture*` if it was. `has_texture()`
  is a `get_texture() != nullptr` check; `verify_texture()` is a
  `load_texture() != nullptr` check (forces a load attempt just to test
  loadability).
- Textures are keyed internally by a `LookupKey` struct (full/alpha
  filename pair + `primary_file_num_channels` + `alpha_file_channel` +
  `TextureType`), not just the filename — the same file loaded with
  different channel/type parameters is tracked as a distinct pool entry.
- **Filter chain:** every load path funnels through `pre_load()` then (after
  the actual disk read) `post_load()`, both of which iterate every
  registered `TexturePoolFilter` (`register_filter()`). `pre_load()` stops
  at the *first* filter that returns non-null (that filter's `Texture` is
  used instead of loading from disk — lets a filter substitute/intercept a
  load entirely); `post_load()` instead chains *every* filter in
  registration order, feeding each filter's output into the next, so
  post-load filters can be composed. Filters loaded automatically at pool
  construction from the repeatable `texture-filter` config var (as
  `lib<name>.so`/`.dll` plugins).
- **Fake texture image:** if `set_fake_texture_image()` (or the
  `fake-texture-image` config var) is set, *every* subsequent load request
  is silently redirected to load that one file instead of the requested
  filename — a global override useful for perf-testing "what if every
  texture were this size/format" without touching content, or for stripping
  actual textures in a shipped debug build.
- Loaded textures are cached to a `BamCache` disk cache when eligible
  (`try_load_cache()`/`tex->set_post_load_store_cache(true)`), separate from
  the in-memory `TexturePool` map — a `BamCache` hit skips the raw
  image-format read entirely on a later run.
- `add_texture()` requires the texture already have a filename set (used as
  its pool key) and unconditionally **replaces** any existing pool entry
  under that filename.
- `release_texture()` only removes the pool's own reference; if nothing
  else holds a `PT(Texture)` to it, the `Texture` is then freed. If never
  called, every texture ever loaded is kept alive forever by the pool.
  `release_texture()` looks the texture up **by its current name**, so
  renaming a `Texture` after loading it can make later release/lookup fail
  to find it.
- `garbage_collect()` is the automatic equivalent of `release_texture()`:
  sweeps every pool entry whose `Texture` has `get_ref_count() == 1`
  (nothing but the pool holds it) and releases those. Not run
  automatically — it is called periodically (e.g. from a task) when
  unused textures need reclaiming without tracking references manually.
- `register_texture_type()`/`get_texture_type()` maintain a
  extension→loader-function registry (`Texture::MakeTextureFunc`) beyond
  what `PNMFileTypeRegistry` (image format plugins) already covers —
  `make_texture(extension)` consults this to construct the right `Texture`
  subclass (e.g. for movie/video textures) before handing off to the
  format reader.
- `get_normalization_cube_map(size)`/`get_alpha_scale_map()` are lazily
  generated, pool-cached procedural textures (via
  `Texture::generate_normalization_cube_map()`/
  `generate_alpha_scale_map()`) shared across the whole process —
  `get_alpha_scale_map()` in particular backs the "apply alpha scale via a
  texture instead of munging vertices" fallback path some GSGs use.

## API

| Signature | Notes |
|---|---|
| `static bool has_texture(const Filename&)` | Already loaded? |
| `static bool verify_texture(const Filename&)` | Force-load-and-check. |
| `static Texture *get_texture(filename[, alpha_filename], primary_file_num_channels=0[, alpha_file_channel=0], read_mipmaps=false)` | Cache-only lookup, no disk I/O; `nullptr` if not loaded. |
| `static Texture *load_texture(filename[, alpha_filename], ..., const LoaderOptions& = {})` `BLOCKING` | Load-or-return-cached. `read_mipmaps` + a `#` in the filename loads a numbered mipmap chain. |
| `static Texture *load_3d_texture/load_2d_texture_array/load_cube_map(filename_pattern, read_mipmaps=false, options={})` `BLOCKING` | Multi-page loads; `#` in the pattern is filled with page index (cube map: 0-5). |
| `static Texture *get_normalization_cube_map(int size)` / `get_alpha_scale_map()` | Shared procedural textures, see behavior notes. |
| `static void add_texture(Texture*)` | Register an already-loaded texture (must have a filename). Replaces any existing entry with the same filename. |
| `static void release_texture(Texture*)` / `release_all_textures()` | Drop the pool's reference(s). |
| `static void rehash()` | Rebuild internal lookup structures (e.g. after filenames changed externally). |
| `static int garbage_collect()` | Release all pool entries with refcount 1; returns count released. |
| `static Texture *find_texture(const string& name)` / `TextureCollection find_all_textures(const string& name = "*")` | Glob-style name search over pooled textures. |
| `static void set_fake_texture_image(const Filename&)` / `clear_fake_texture_image()` / `has_fake_texture_image()` / `get_fake_texture_image()` | Global load-redirect override, see behavior notes. |
| `static PT(Texture) make_texture(const string& extension)` | Construct the right `Texture` subclass for a file extension via the type registry. |
| `static void write(ostream&)` / `static void list_contents(ostream& \| default)` | Debug dump of pool contents. |
| `void register_texture_type(MakeTextureFunc*, const string& extensions)` | Public but non-`PUBLISHED` (C++-only); register a custom loader for one or more extensions. |
| `void register_filter(TexturePoolFilter*)` | Register a load-intercepting filter, see behavior notes. |

## Usage

```cpp
PT(Texture) tex = TexturePool::load_texture("maps/brick.jpg");
if (tex == nullptr) {
  // load failed
}
```

## See also

- [Texture](Texture.md), [TextureCollection](TextureCollection.md),
  [TexturePoolFilter](TexturePoolFilter.md)
