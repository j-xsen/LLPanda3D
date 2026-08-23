# ShaderContext

**Source:** `panda/src/gobj/shaderContext.h` (+ `.I`, `.cxx`)
**Inherits:** [SavedContext](SavedContext.md) **Inherited by:** GSG-backend-specific subclasses (e.g. GLSL/Cg-under-GL contexts; not part of `gobj`)

The GSG-side compiled/linked handle for a [Shader](Shader.md) — same
"prepared object" role as `TextureContext`/`VertexBufferContext`, see the
module README's "`PreparedGraphicsObjects` / `*Context` handshake"
section. `ShaderContext` itself is a near-empty abstract base: "the
languages are so different and the graphics APIs have so little in
common [that] the base class contains almost nothing" — every method
besides `get_shader()` is a virtual no-op stub, entirely overridden by
each backend's own subclass (Cg-under-GL, GLSL-under-GL, HLSL-under-DX,
etc., none of which live in `gobj`).

## Behavior notes

- Every virtual (`bind()`, `unbind()`, `issue_parameters()`,
  `update_shader_vertex_arrays()`, `update_shader_texture_bindings()`,
  `update_shader_buffer_bindings()`, …) defaults to a no-op or trivial
  return in the base class — this is a pure extension-point class, not
  one with meaningful shared behavior of its own. Reading this header
  alone tells you the *shape* of the shader-binding lifecycle (set
  state/transform → bind → issue parameters → update vertex/texture/
  buffer bindings → unbind) but none of the actual work; that's all in
  the per-backend subclass.
- `uses_standard_vertex_arrays()` defaults `true` / `uses_custom_vertex_arrays()`
  defaults `false` — a backend overrides these together when its shader
  variant wants to supply vertex data through a nonstandard path instead
  of the usual `GeomVertexData` column bindings.

## API

| Signature | Notes |
|---|---|
| `ShaderContext(Shader *se)` | Stores the owning `Shader` in `_shader`. |
| `Shader *get_shader() const` | Property `shader`. |
| `virtual bool valid()` | Base returns `false`; a real backend context reports `true` once successfully compiled/linked. |
| `virtual void set_state_and_transform(const RenderState *, const TransformState *, const TransformState *, const TransformState *)` | Backend hook: push current render/transform state into the shader's parameter specs. |
| `virtual void bind()` / `virtual void unbind()` | Make/unmake this program current on the GSG. |
| `virtual void issue_parameters(int altered)` | Re-push only the uniform/parameter specs affected by the `altered` `ShaderStateDep` bitmask. |
| `virtual void disable_shader_vertex_arrays()` / `virtual bool update_shader_vertex_arrays(ShaderContext *prev, bool force)` | Vertex-attribute binding lifecycle. |
| `virtual void disable_shader_texture_bindings()` / `virtual void update_shader_texture_bindings(ShaderContext *prev)` | Texture-sampler binding lifecycle. |
| `virtual void update_shader_buffer_bindings(ShaderContext *prev)` | `ShaderBuffer` (SSBO-style) binding lifecycle. |

## See also

- [Shader](Shader.md) — the compiled-from source this wraps
- [SavedContext](SavedContext.md) — shared base of every `*Context` class
- [PreparedGraphicsObjects](PreparedGraphicsObjects.md) — owns/tracks these
