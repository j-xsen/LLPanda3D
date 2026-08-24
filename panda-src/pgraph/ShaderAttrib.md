# ShaderAttrib

**Source:** `panda/src/pgraph/shaderAttrib.h` (+ `.I`, `.cxx`; excludes
`shaderAttrib_ext.h/.cxx`, Python-only glue for `set_shader_input`/
`set_shader_inputs` accepting `PyObject*`)
**Inherits:** RenderAttrib

Carries a node's `Shader` selection (explicit shader, auto-generated
shader, or none), per-name `ShaderInput` bindings passed to that shader,
instancing count, and a handful of boolean render flags. One of the more
complex attribs — it accumulates state incrementally down the scene graph
by merging inputs/flags rather than replacing them outright on compose.
`Shader`/`ShaderInput` are also defined in `panda/src/pgraph`
([ShaderInput](ShaderInput.md), documented separately in this module);
`ShaderPool` caches loaded `Shader` objects.

## Behavior notes

- Three shader states: **explicit** (`set_shader(s, priority)` —
  `_has_shader=true`, `_auto_shader=false`), **auto**
  (`set_shader_auto(priority)` — `_has_shader=true`, `_auto_shader=true`,
  `_shader=nullptr`, requests Panda's automatic shader generator), and
  **unset** (`clear_shader()` — `_has_shader=false`, inherits from a
  parent node's `ShaderAttrib` if any). `set_shader_off()` is a fourth,
  explicit "no shader" state.
- `set_shader_auto(BitMask32 shader_switch, priority)` toggles individual
  auto-shader features via `Shader::bit_AutoShaderNormal/Glow/Gloss/Ramp/
  Shadow` bits (`auto_normal_on()` etc.) — the plain `set_shader_auto()`
  enables all five; `Shader` itself lives in `panda/src/pgraph` too but is
  otherwise undocumented here (see `shader.h`).
- **Compose semantics (`compose_impl`)** — this is the important part for
  understanding how shader state accumulates down a scene graph:
  - Shader selection: the downstream (`other`) attrib's shader wins only
    if it *has* a shader and its priority is `>=` the current one's — a
    lower-priority explicit shader on a child node does **not** override
    a higher-priority one set higher in the graph.
  - Shader inputs: merged key-by-key (`_inputs` is a
    `pmap<InternalName*, ShaderInput>`) — an input present in both is kept
    from whichever side has the higher-or-equal per-input priority
    (`ShaderInput::get_priority()`), not just "downstream wins."
  - Instance count: if the accumulated count is `0` (unset), take the
    downstream value; otherwise only overwrite if downstream's count is
    `> 0` (an explicit "no instancing" downstream doesn't erase an
    instance count set higher up).
  - Flags (`F_disable_alpha_write`, `F_subsume_alpha_test`,
    `F_hardware_skinning`, `F_shader_point_size`): tracked as a `_flags`/
    `_has_flags` bitmask pair so "flag explicitly set to false" is
    distinguishable from "flag not mentioned" — compose clears
    accumulated bits that downstream explicitly sets, then ORs in
    downstream's bits, so downstream always wins for any flag it touches.
- `register_with_read_factory()` is a stub (`// IMPLEMENT ME`) —
  `ShaderAttrib` is **not** Bam-serializable in this version; shaders set
  in code aren't saved/restored via `.bam` files.
- `get_shader_input_vector/texture/ptr/matrix/buffer(id)` are typed
  accessors that assert/error if the stored `ShaderInput` isn't actually
  that type — check `has_shader_input()` first if unsure.

## API

| Signature | Notes |
|---|---|
| `static CPT(RenderAttrib) make(const Shader *shader=nullptr, int priority=0)` | |
| `static CPT(RenderAttrib) make_off()` / `make_default()` | |
| `enum { F_disable_alpha_write, F_subsume_alpha_test, F_hardware_skinning, F_shader_point_size }` | Flag bit indices |
| `bool has_shader() const` / `auto_shader() const` / `int get_shader_priority() const` | |
| `int get_instance_count() const` | 0 = no instancing |
| `bool auto_normal_on/auto_glow_on/auto_gloss_on/auto_ramp_on/auto_shadow_on() const` | Per-feature auto-shader toggles |
| `CPT(RenderAttrib) set_shader(const Shader*, priority=0) const` / `set_shader_off(priority=0) const` / `set_shader_auto(priority=0) const` / `set_shader_auto(BitMask32, priority=0) const` / `clear_shader() const` | |
| `CPT(RenderAttrib) set_shader_input(const ShaderInput &) const` (+ typed overloads for Texture/NodePath/PTA_float/PTA_double/matrices/vectors/scalars) | |
| `CPT(RenderAttrib) set_instance_count(int) const` | |
| `CPT(RenderAttrib) set_flag(int flag, bool value) const` / `clear_flag(int flag) const` | |
| `CPT(RenderAttrib) clear_shader_input(const InternalName* / std::string&) const` / `clear_all_shader_inputs() const` | |
| `bool get_flag(int flag) const` / `has_shader_input(CPT_InternalName) const` | |
| `const Shader *get_shader() const` | |
| `const ShaderInput &get_shader_input(const InternalName* / std::string&) const` | |
| `const NodePath &get_shader_input_nodepath(const InternalName*) const` / `LVecBase4 get_shader_input_vector(InternalName*) const` / `Texture *get_shader_input_texture(const InternalName*, SamplerState* sampler=nullptr) const` / `const Shader::ShaderPtrData *get_shader_input_ptr(const InternalName*) const` / `const LMatrix4 &get_shader_input_matrix(const InternalName*, LMatrix4 &matrix) const` / `ShaderBuffer *get_shader_input_buffer(const InternalName*) const` | Typed input accessors |

## Usage

```cpp
PT(Shader) shader = Shader::load(Shader::SL_GLSL, "vert.glsl", "frag.glsl");
node_path.set_shader(shader);
node_path.set_shader_input("light_pos", LVecBase3(0, 0, 10));
node_path.set_shader_input("tex", my_texture);
node_path.set_shader_auto(); // let Panda pick a shader based on the RenderState
```

## See also

[README — the state pipeline](README.md#the-state-pipeline),
[RenderAttrib](RenderAttrib.md), [ShaderInput](ShaderInput.md),
[ShaderPool](ShaderPool.md)
