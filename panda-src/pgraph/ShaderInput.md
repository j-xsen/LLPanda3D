# ShaderInput

**Source:** `panda/src/pgraph/shaderInput.h` (+ `.I`, `.cxx`)
**Inherits:** (standalone value type)

A small tagged-union container binding one named value — a texture,
`NodePath`, numeric array, vector, matrix, shader storage buffer, or
arbitrary `ParamValueBase` — to a shader input name. A set of `ShaderInput`s
attached to a node (via [ShaderAttrib](ShaderAttrib.md)) is how application
code feeds `uniform`/`in` values into a GLSL/Cg shader without going through
the fixed-function `RenderAttrib` pipeline (materials, lights, textures).

## Behavior notes

- Default-constructible (`ShaderInput() = default`) with type `M_invalid`;
  `get_blank()` returns a shared static instance of this for use as a
  "no input" sentinel — `operator bool()` is `false` for it.
- Which storage member is live depends on `_type`: `M_vector`/`M_numeric`
  inputs pack into `_stored_vector`/`_stored_ptr` (a `Shader::ShaderPtrData`,
  which itself remembers pointer + element count + numeric type — float vs.
  double vs. int); everything else (`M_texture`, `M_nodepath`, `M_param`,
  `M_texture_sampler`, `M_texture_image`, `M_buffer`) boxes into the generic
  `PT(TypedWritableReferenceCount) _value` slot (as a `Texture`,
  `ParamNodePath`, `ParamTextureSampler`, `ParamTextureImage`, or
  `ShaderBuffer`).
- `LVecBase{2,3,4}{f,d,i}` vector/matrix overloads all normalize down to
  `PN_stdfloat`-precision storage in `_stored_vector` (4-component,
  zero-padded) *and* keep the original precision in `_stored_ptr` — so
  `get_vector()` is always a `PN_stdfloat` `LVecBase4`, while `get_ptr()`
  preserves the original type/precision for the GSG to consult.
  `LVecBase*i` (int) vectors are the exception: only `_stored_vector`
  (cast to float) is meaningful for them via `get_vector()`.
  `PTA_*` (dynamic pointer-to-array) inputs are always classified
  `M_numeric`, not `M_vector` — `M_vector` is reserved for fixed-size
  `LVecBase*`/`LMatrix*` literal overloads.
- `get_texture()` accepts any texture-flavored type (`M_texture`,
  `M_texture_sampler`, `M_texture_image`) and unwraps accordingly; for any
  other type it returns `nullptr` rather than asserting.
- `get_nodepath()` has **no type check** — calling it when
  `get_value_type() != M_nodepath` is documented UB ("will crash").
- Comparison (`==`, `!=`, `<`) and `add_hash()` compare by `_type`+`_name`+
  `_priority` first, then by the *pointer* value of `_stored_ptr`/`_value`
  for pointer-backed types (not deep content) — two inputs wrapping
  equal-content-but-different-instance data are NOT equal.
- `priority` (constructor's last arg) is used by `ShaderAttrib`/`RenderState`
  composition to decide which of two same-named inputs from different
  attribs down the node chain wins (higher priority overrides).
- `register_with_read_factory()` is an unimplemented stub ("IMPLEMENT ME")
  — `ShaderInput` is not currently bam-serializable on its own.

## API

| Constructor | Notes |
|---|---|
| `ShaderInput(name, int priority=0)` | Invalid/blank input with just a name |
| `ShaderInput(name, Texture *tex, int priority=0)` | Plain texture bind |
| `ShaderInput(name, Texture *tex, bool read, bool write, int z=-1, int n=0, int priority=0)` | Image-load-store binding (`M_texture_image`) |
| `ShaderInput(name, Texture *tex, const SamplerState &sampler, int priority=0)` | Texture with explicit custom sampler |
| `ShaderInput(name, const NodePath &np, int priority=0)` | Bind a NodePath (e.g. for `wrt()`-relative matrix computation in shader) |
| `ShaderInput(name, ParamValueBase *param, int priority=0)` | Generic boxed param |
| `ShaderInput(name, ShaderBuffer *buf, int priority=0)` | Shader storage buffer |
| `ShaderInput(name, const PTA_float/double/int &ptr, int priority=0)` | Dynamic numeric array |
| `ShaderInput(name, const PTA_LVecBase{2,3,4}{f,d,i}/LMatrix{3,4}{f,d} &ptr, int priority=0)` | Dynamic array of vectors/matrices |
| `ShaderInput(name, const LVecBase{2,3,4}{f,d,i}/LMatrix{3,4}{f,d} &val, int priority=0)` | Single literal vector/matrix |

| Method | Notes |
|---|---|
| `get_name()` → `const InternalName *` | |
| `get_value_type()` → `int` (`ShaderInputType`) | `M_invalid`, `M_texture`, `M_nodepath`, `M_vector`, `M_numeric`, `M_texture_sampler`, `M_param`, `M_texture_image`, `M_buffer` |
| `get_priority()` → `int` | |
| `get_vector()` → `const LVecBase4 &` | Valid for `M_vector`/`M_numeric` inputs |
| `get_ptr()` → `const Shader::ShaderPtrData &` | Precision/count-preserving raw data |
| `get_texture()` → `Texture *` | `nullptr` if not texture-flavored |
| `get_sampler()` → `const SamplerState &` | Falls back to the texture's default sampler, or `SamplerState::get_default()` |
| `get_nodepath()` → `const NodePath &` | **Unchecked** — only valid if `M_nodepath` |
| `get_param()` → `ParamValueBase *` | |
| `operator bool()` | `false` iff `M_invalid` |
| `operator ==`/`!=`/`<` | See Behavior notes |
| `add_hash(size_t)` → `size_t` | |
| `AccessFlags` enum | `A_read`, `A_write`, `A_layered` — used by the image-binding constructor |

## Usage

```cpp
PT(Texture) noise_tex = TexturePool::load_texture("noise.png");
NodePath np = ...;

CPT(RenderAttrib) sattr = ShaderAttrib::make();
sattr = DCAST(ShaderAttrib, sattr)->set_shader(
    Shader::load(Shader::SL_GLSL, "v.vert", "f.frag"));
sattr = DCAST(ShaderAttrib, sattr)->set_shader_input(
    ShaderInput("noise", noise_tex));
sattr = DCAST(ShaderAttrib, sattr)->set_shader_input(
    ShaderInput("model", np));
np.set_attrib(sattr);
```

## See also

- [ShaderAttrib](ShaderAttrib.md) — carries a named set of `ShaderInput`s on a node
- [ShaderPool](ShaderPool.md) — loads/caches the `Shader` object itself
