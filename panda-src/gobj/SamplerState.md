# SamplerState

**Source:** `panda/src/gobj/samplerState.h` (+ `.I`, `.cxx`)
**Inherits:** (none — plain value type) **Inherited by:** (none)

A small value-type bundle of texture-sampling settings (wrap mode per
axis, min/mag filter, anisotropic degree, border color, LOD range/bias).
Every `Texture` carries a default `SamplerState`, but a `TextureStage`/
`ShaderInput` binding can supply a different one to sample the *same*
texture differently in different places (e.g. one stage repeating, another
clamped) without duplicating the `Texture` itself. Deliberately packed to
fit in 32 bytes (bitfield-packed enums) since scenes may hold many.

## Behavior notes

- **Defaults:** `WM_repeat` on all three wrap axes, `FT_default` for both
  filters (resolves through `get_effective_minfilter()`/
  `get_effective_magfilter()` to the `texture-minfilter`/
  `texture-magfilter` config vars, which themselves default per texture
  format), `min_lod`/`max_lod` of `-1000`/`1000` (effectively unbounded),
  `lod_bias` 0, `anisotropic_degree` 0 (meaning "use the
  `texture-anisotropic-degree` config default" — see
  `get_effective_anisotropic_degree()`; explicitly set `1` to force
  anisotropic filtering *off* rather than defaulting).
- `uses_mipmaps()` / `is_mipmap(FilterType)` — only `minfilter` can request
  mipmapping (`FT_*_mipmap_*` values); `magfilter` never does, since
  magnification means the texture is being viewed larger than native
  resolution, where mipmaps are irrelevant.
- `FT_shadow` is not really a filter mode in the GL/D3D sense — it maps to
  the GL `ARB_shadow` depth-comparison sampling extension (hardware PCF
  shadow map sampling), grouped into `FilterType` as a convenience.
- Comparison (`operator==`/`!=`/`<`) goes through `compare_to()`, making
  `SamplerState` usable as a map/set key or dedup key — this is how the
  GSG can cache one `SamplerContext` per distinct `SamplerState` rather
  than per texture (see [SamplerContext](SamplerContext.md)).
- `get_default()` returns a reference to a shared static default instance
  — useful for comparing "is this the default sampler" without
  constructing a temporary.

## API

| Signature | Notes |
|---|---|
| `SamplerState()` | Default-constructs to the defaults above. |
| `static const SamplerState &get_default()` | Shared default instance. |
| `void set_wrap_u/v/w(WrapMode)` / `WrapMode get_wrap_u/v/w() const` | Per-axis wrap mode (`WM_clamp`, `WM_repeat`, `WM_mirror`, `WM_mirror_once`, `WM_border_color`); `w` only meaningful for 3-D textures. |
| `void set_minfilter/magfilter(FilterType)` / `FilterType get_minfilter/magfilter() const` | See `FilterType` values below. |
| `FilterType get_effective_minfilter/magfilter() const` | Resolves `FT_default` through the config-var default. |
| `void set_anisotropic_degree(int)` / `int get_anisotropic_degree() const` / `int get_effective_anisotropic_degree() const` | 0 = use config default; 1 = explicitly off; >1 = explicit degree. |
| `void set_border_color(LColor)` / `const LColor &get_border_color() const` | Used only for `WM_border_color` clamp color in Panda (not full GL border texel support). |
| `void set_min_lod/max_lod/lod_bias(PN_stdfloat)` / getters | Mipmap LOD clamp range and bias. |
| `bool uses_mipmaps() const` | True if `get_effective_minfilter()` is a mipmap variant. |
| `static bool is_mipmap(FilterType)` | Static classification helper. |
| `static string format_filter_type(FilterType)` / `static FilterType string_filter_type(string)` | Name ⇄ enum, also used by `operator<<`/`>>`. |
| `static string format_wrap_mode(WrapMode)` / `static WrapMode string_wrap_mode(string)` | Same for `WrapMode`. |
| `bool operator==/!=/< (const SamplerState &) const` | Via `compare_to()`. |
| `void prepare(PreparedGraphicsObjects *) const` / `bool is_prepared(...) const` / `void release(...) const` / `SamplerContext *prepare_now(PreparedGraphicsObjects *, GraphicsStateGuardianBase *) const` | Same GSG-preparation handshake as `Texture`/`Shader` — see [SamplerContext](SamplerContext.md). |

**`FilterType` values:** `FT_nearest`, `FT_linear` (mag/min), plus
min-only `FT_nearest_mipmap_nearest`, `FT_linear_mipmap_nearest`,
`FT_nearest_mipmap_linear`, `FT_linear_mipmap_linear` (bilinear-across-
mipmap-levels = "trilinear"), `FT_shadow`, `FT_default`, `FT_invalid`.

**`WrapMode` values:** `WM_clamp`, `WM_repeat`, `WM_mirror`,
`WM_mirror_once`, `WM_border_color`, `WM_invalid`.

## Usage

```cpp
SamplerState sampler;
sampler.set_wrap_u(SamplerState::WM_clamp);
sampler.set_wrap_v(SamplerState::WM_clamp);
sampler.set_minfilter(SamplerState::FT_linear_mipmap_linear);
node_path.set_texture(tex, sampler);  // NodePath convenience overload
```

## See also

- [SamplerContext](SamplerContext.md) — GSG-side prepared handle for a
  `SamplerState`
- [Texture](Texture.md), [TextureStage](TextureStage.md)
