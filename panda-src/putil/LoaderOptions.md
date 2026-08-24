# LoaderOptions

**Source:** `panda/src/putil/loaderOptions.h` / `.I` / `.cxx`
**Inherits:** (none)
**Inherited by:** (none)

A plain value type bundling the flags passed to Panda's model/resource
loader (`Loader`, outside this module) — whether to search the model path,
report errors, convert animation data, bypass caches, and a parallel set of
texture-specific flags. Constructed with sensible defaults
(`LF_search | LF_report_errors`) and typically tweaked before a single
`loader->load_sync()`/`load_async()` call.

## Behavior notes

- **`LF_convert_anim` is a combined bit** (`LF_convert_skeleton |
  LF_convert_channels`), not an independent flag — `output()` checks for the
  combined value first and prints `LF_convert_anim` if both bits are set,
  falling back to printing the two component flags separately otherwise.
  `LF_no_cache` is the same pattern over `LF_no_disk_cache | LF_no_ram_cache`.
- **The constructor seeds texture flags from prc config variables**
  (`preload-textures`, `preload-simple-textures`, `compressed-textures`),
  read once via function-local `static` pointers to sidestep static-init
  ordering issues — so `LoaderOptions()`'s default `texture_flags` already
  reflects global config, not just zero.
- **`auto_texture_scale` defaults to `ATS_unspecified`**, meaning "don't
  override — use the global default," distinct from explicitly requesting
  `ATS_none`.
- There's a `constexpr` two-argument constructor
  (`LoaderOptions(int flags, int texture_flags)`) alongside the normal
  runtime constructor — it skips the prc-variable lookups entirely (leaves
  `texture_flags` exactly as passed, `auto_texture_scale` at
  `ATS_unspecified`), intended for compile-time-constant option sets.
- `output()` prints `"0"` for either flag group if no bits are set, rather
  than an empty string — the format is always `LoaderOptions(<flags>, <texture_flags>[, ATS_<scale>])`.

## API

```cpp
enum LoaderFlags {
  LF_search            = 0x0001,
  LF_report_errors     = 0x0002,
  LF_convert_skeleton  = 0x0004,
  LF_convert_channels  = 0x0008,
  LF_convert_anim      = 0x000c,  // skeleton + channels
  LF_no_disk_cache     = 0x0010,  // disallow BamCache
  LF_no_ram_cache      = 0x0020,  // disallow ModelPool
  LF_no_cache          = 0x0030,  // no_disk + no_ram
  LF_cache_only        = 0x0040,  // fail if not already cached
  LF_allow_instance    = 0x0080,  // returned pointer may be a shared instance
};

enum TextureFlags {
  TF_preload           = 0x0004,  // force a RAM image
  TF_preload_simple    = 0x0008,  // force a simple (low-res placeholder) RAM image
  TF_allow_1d          = 0x0010,  // Nx1 images become 1-d textures
  TF_generate_mipmaps  = 0x0020,
  TF_multiview         = 0x0040,  // load as a multiview texture (see texture_num_views)
  TF_integer           = 0x0080,  // load as an integer texture
  TF_float             = 0x0100,  // load as a floating-point (depth) texture
  TF_allow_compression = 0x0200,
};
```

| Signature | Notes |
|---|---|
| `LoaderOptions(int flags = LF_search\|LF_report_errors)` | Also seeds texture_flags from prc vars |
| `constexpr LoaderOptions(int flags, int texture_flags)` | No prc-variable lookup |
| `void set_flags(int)` / `int get_flags() const` | |
| `void set_texture_flags(int)` / `int get_texture_flags() const` | |
| `void set_texture_num_views(int)` / `int get_texture_num_views() const` | Only meaningful with `TF_multiview`; disambiguates z-levels vs. views for 3-d/array textures |
| `void set_auto_texture_scale(AutoTextureScale)` / `get_auto_texture_scale() const` | Default `ATS_unspecified` |
| `void output(std::ostream&) const` | Also via `operator<<` |

## Usage

```cpp
LoaderOptions opts(LoaderOptions::LF_search |
                    LoaderOptions::LF_report_errors |
                    LoaderOptions::LF_convert_anim);
opts.set_auto_texture_scale(ATS_down);
```

## See also

[AutoTextureScale.md](AutoTextureScale.md) · [README.md](README.md)
