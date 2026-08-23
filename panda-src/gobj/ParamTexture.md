# ParamTextureSampler / ParamTextureImage

**Source:** `panda/src/gobj/paramTexture.h` (+ `.I`, `.cxx`)
**Inherits:** both `ParamValueBase` (external/undocumented, from `putil` — a `TypedWritableReferenceCount` shader-parameter wrapper base)

Two small, tightly-coupled classes documented together since both exist
purely to bundle a [Texture](Texture.md) with binding metadata for shader
input purposes (`ShaderInput`, see `pgraph`), and neither has much surface
area on its own.

## ParamTextureSampler

Bundles a `Texture` with an explicit [SamplerState](SamplerState.md),
letting a shader input override the texture's own default sampling
parameters (wrap mode, filter, border color) without mutating the shared
`Texture` object itself — useful when the same texture needs different
sampling behavior in different shader inputs/contexts.

| Signature | Notes |
|---|---|
| `ParamTextureSampler(Texture *tex, const SamplerState &sampler)` | |
| `Texture *get_texture() const` | Also `MAKE_PROPERTY(texture, ...)`. |
| `const SamplerState &get_sampler() const` | Also `MAKE_PROPERTY(sampler, ...)`. |
| `virtual TypeHandle get_value_type() const` | `ParamValueBase` type-dispatch hook. |

## ParamTextureImage

Bundles a `Texture` with read/write access flags and a specific mip
level/array-layer binding, for binding a texture as a **shader image**
(direct read/write GPU image access from a compute/fragment shader, as
opposed to normal sampled texture access) — an "esoteric feature" per the
source comment, mainly used with compute shaders.

- `_access` is a bitmask (`A_read=0x01`, `A_write=0x02`, `A_layered=0x04`),
  packed into a 4-bit field; `_bind_level`/`_bind_layer` are packed 8-bit/
  20-bit fields — this is a deliberately compact struct since many of these
  may exist per shader/material.
- `get_bind_layered()` (the `A_layered` flag) selects whether *all* array
  layers of the texture are bound simultaneously (for a texture array/3D
  texture), vs. `get_bind_layer()` selecting one specific layer/depth
  slice when not layered.
- Constructor default `z=-1` combined with `n=0` — check the concrete
  binding-layer semantics against the GSG backend if `z` matters; `-1`
  reads as "not specified" given the field is otherwise a real layer index.

| Signature | Notes |
|---|---|
| `ParamTextureImage(Texture *tex, bool read, bool write, int z=-1, int n=0)` | `z` = bind layer, `n` = bind mip level. |
| `Texture *get_texture() const` | Also `MAKE_PROPERTY(texture, ...)`. |
| `bool has_read_access() const` / `bool has_write_access() const` | Also `MAKE_PROPERTY(read_access, ...)` / `MAKE_PROPERTY(write_access, ...)`. |
| `bool get_bind_layered() const` | Bind all layers vs. one. |
| `int get_bind_level() const` | Mip level; also `MAKE_PROPERTY(bind_level, ...)`. |
| `int get_bind_layer() const` | Specific layer when not layered; also `MAKE_PROPERTY2(bind_layer, get_bind_layered, get_bind_layer)`. |
| `virtual TypeHandle get_value_type() const` | |

Both classes support the standard Bam file read/write hooks
(`register_with_read_factory`/`write_datagram`/`complete_pointers`/
`fillin`) for serialization, same pattern as other `TypedWritable`s in
this reference — not detailed further here.

## See also

- [Texture](Texture.md), [SamplerState](SamplerState.md)
- `ShaderInput`, `ShaderAttrib` (pgraph, see
  [../pgraph/README.md](../pgraph/README.md)) — where these get consumed
