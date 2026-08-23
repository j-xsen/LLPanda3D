# ShaderBuffer

**Source:** `panda/src/gobj/shaderBuffer.h` (+ `.I`, `.cxx`)
**Inherits:** TypedWritableReferenceCount, Namable, [GeomEnums](README.md#shared-enums-geomenums) **Inherited by:** (none)

*Since 1.10.0.* A generic raw buffer that lives in graphics memory —
bound to a shader as a storage/uniform buffer object (SSBO-style), rather
than as a texture or a `GeomVertexData` array. Construct with either a
byte size (uninitialized/GPU-only content) or an initial `vector_uchar`
payload, tag it with a `GeomEnums::UsageHint`, then bind it through a
`ShaderAttrib`/`ShaderInput` (`pgraph`) the same way a texture input is
bound.

## Behavior notes

- Follows the exact same `prepare()`/`is_prepared()`/`prepare_now()`/
  `release()`/`release_all()` GSG-preparation lifecycle as `Texture` and
  `Shader` (see the module README's "`PreparedGraphicsObjects` /
  `*Context` handshake"), producing a plain `BufferContext` rather than a
  dedicated subclass — a `ShaderBuffer`'s uploaded state is just "a GPU
  buffer," with no extra bookkeeping beyond what `BufferContext` already
  provides. `_contexts` is lazily allocated (`nullptr` until first
  prepared) and freed back to `nullptr` once empty, unlike `Shader`'s
  always-present map.
- The destructor calls `release_all()` — a `ShaderBuffer` going out of
  scope automatically frees its GPU-side buffers on every GSG it was
  prepared on.
- Bam serialization pads `_initial_data` up to a 16-byte boundary on
  read-in (`(_data_size_bytes + 15u) & ~15u`) even though only
  `_data_size_bytes` bytes are meaningful — a alignment convenience for
  backends, not a content change.
- Default-constructible only via the private no-arg constructor used by
  `make_from_bam()`; application code always uses one of the two
  `PUBLISHED` constructors.

## API

| Signature | Notes |
|---|---|
| `ShaderBuffer(string name, uint64_t size, UsageHint usage_hint)` | Uninitialized GPU buffer of `size` bytes. |
| `ShaderBuffer(string name, vector_uchar initial_data, UsageHint usage_hint)` | Buffer pre-populated with `initial_data`. |
| `uint64_t get_data_size_bytes() const` | Property `data_size_bytes`. |
| `UsageHint get_usage_hint() const` | Property `usage_hint`; see [GeomEnums](README.md#shared-enums-geomenums). |
| `const unsigned char *get_initial_data() const` | Raw pointer to the initial payload, if any. |
| `void prepare(PreparedGraphicsObjects *)` | Queue async upload. |
| `bool is_prepared(PreparedGraphicsObjects *) const` | Already uploaded or queued? |
| `BufferContext *prepare_now(PreparedGraphicsObjects *, GraphicsStateGuardianBase *)` | Synchronous upload; normally GSG-internal. |
| `bool release(PreparedGraphicsObjects *)` | Free on one GSG. |
| `int release_all()` | Free on every GSG; returns count freed. |

## Usage

```cpp
PT(ShaderBuffer) buf = new ShaderBuffer(
  "particles", 1024 * 1024, GeomEnums::UH_dynamic);
node_path.set_shader_input("ParticleBuffer", buf);
```

## See also

- [Shader](Shader.md), [ShaderContext](ShaderContext.md)
- [BufferContext](BufferContext.md) — the `*Context` type this prepares to
- [PreparedGraphicsObjects](PreparedGraphicsObjects.md)
