# TextureStage

**Source:** `panda/src/gobj/textureStage.h` (+ `.I`, `.cxx`)
**Inherits:** TypedWritableReferenceCount

Defines the properties of one named "slot" in the multitexture pipeline: a
`TextureAttrib` (`pgraph`) associates a `Texture` with each of a set of
`TextureStage`s applied to a piece of geometry, and the GSG sorts and
renders all currently-active stages in `sort` order, combining each one
into the running result per its `Mode`/combine settings. One `TextureStage`
is created per texture "role" needed (diffuse, normal map, gloss,
…) and reused across many `TextureAttrib`s/`NodePath`s — unlike
`RenderAttrib`, `TextureStage` is **not** interned/immutable; it's a
regular mutable ref-counted object, so mutating a shared stage affects
every `TextureAttrib` that references it.

## Behavior notes

- Default-constructed state: `sort=0`, `priority=0`,
  `texcoord_name=InternalName::get_texcoord()`, `mode=M_modulate`,
  `color=(0,0,0,1)`, `rgb_scale=alpha_scale=1`, `saved_result=false`,
  `tex_view_offset=0`.
- `set_sort()`/`set_priority()` both bump a **global** `UpdateSeq`
  (`_sort_seq`, shared across every `TextureStage` in the process) —
  `TextureAttrib` compares its cached copy against
  `TextureStage::get_sort_seq()` to know when it must re-sort its internal
  stage list, so any stage's sort/priority change invalidates every
  `TextureAttrib`'s sort cache, not just attribs referencing that stage.
- `sort` controls render *order* (lowest first); `priority` controls which
  stages get *dropped* when the hardware supports fewer simultaneous
  texture units than are requested — these are independent knobs.
- `is_fixed_function()` is defined as `mode < M_normal`: every `Mode` enumerator
  before `M_normal` (`M_modulate`, `M_decal`, `M_blend`, `M_replace`,
  `M_add`, `M_combine`, `M_blend_color_scale`, `M_modulate_glow`,
  `M_modulate_gloss`) is usable by the classic fixed-function pipeline;
  everything from `M_normal` onward (`M_normal_height`, `M_glow`, `M_gloss`,
  `M_height`, `M_selector`/`M_metallic_roughness`, `M_normal_gloss`,
  `M_emission`) only makes sense to a shader.
- `set_combine_rgb()`/`set_combine_alpha()` are overloaded by operand count
  (1/2/3 source-operand pairs) and each overload asserts (`nassertv`) that
  the given `CombineMode` expects exactly that many operands and that each
  `CombineOperand` is legal for RGB vs. alpha respectively (some operands,
  e.g. dot3 modes, are RGB-only). Calling any `set_combine_*` overload
  forces `mode` to `M_combine` as a side effect.
- `get_tangent_name()`/`get_binormal_name()` are derived, not stored: they
  take the stage's `texcoord_name` and substitute `"tangent"`/`"binormal"`
  as the first path component (or return the plain
  `InternalName::get_tangent()`/`get_binormal()` if `texcoord_name` has no
  parent) — so setting a custom UV set name automatically implies matching
  tangent/binormal column names, per Panda's `InternalName` hierarchical
  naming convention.
- `update_color_flags()` (called after any mode/combine-source change)
  recomputes four cached booleans (`uses_color`, `involves_color_scale`,
  `uses_primary_color`, `uses_last_saved_result`) by scanning all combine
  sources for `CS_constant`/`CS_constant_color_scale`/`CS_primary_color`/
  `CS_last_saved_result` — these let `TextureAttrib`/the auto-shader
  generator quickly check whether a stage cares about the current color
  scale or primary color without re-scanning combine state each frame.
- `mark_used_by_auto_shader()` sets a `mutable` flag; from then on, *any*
  further mutation to that stage (`set_sort`, `set_priority`,
  `set_texcoord_name`, `set_rgb_scale`, `set_alpha_scale`,
  `set_saved_result`, any combine setter) additionally calls
  `GraphicsStateGuardianBase::mark_rehash_generated_shaders()` — this is how
  the shader auto-generator (`ShaderAttrib`'s implicit-shader path) knows to
  regenerate previously-generated shaders when a stage they depend on
  changes.
- `get_default()` lazily allocates a single shared `TextureStage("default")`
  the first time it's called; this is the implicit stage `TextureAttrib`
  uses when an app binds a `Texture` without naming a stage.
- Comparison (`operator==`/`<`, `compare_to()`) is a full member-wise
  comparison of every field (name, sort, priority, texcoord name pointer,
  mode, color, scales, all combine fields) — used by `TextureAttrib` to
  deduplicate/order its stage list, not just identity comparison.

## API

| Signature | Notes |
|---|---|
| `explicit TextureStage(const std::string &name)` | |
| `TextureStage(const TextureStage &copy)` / `operator=` | Full member-wise copy. |
| `enum Mode` | `M_modulate, M_decal, M_blend, M_replace, M_add, M_combine, M_blend_color_scale, M_modulate_glow, M_modulate_gloss` (fixed-function) then `M_normal, M_normal_height, M_glow, M_gloss, M_height, M_selector`(`=M_metallic_roughness`)`, M_normal_gloss, M_emission` (shader-only). |
| `enum CombineMode` | `CM_undefined, CM_replace, CM_modulate, CM_add, CM_add_signed, CM_interpolate, CM_subtract, CM_dot3_rgb, CM_dot3_rgba` (dot3 variants RGB-only). |
| `enum CombineSource` | `CS_undefined, CS_texture, CS_constant, CS_primary_color, CS_previous, CS_constant_color_scale, CS_last_saved_result`. |
| `enum CombineOperand` | `CO_undefined, CO_src_color, CO_one_minus_src_color, CO_src_alpha, CO_one_minus_src_alpha`. |
| `set_name/get_name`, `set_sort/get_sort`, `set_priority/get_priority` | |
| `set_texcoord_name(InternalName* \| string)/get_texcoord_name` | |
| `get_tangent_name() / get_binormal_name()` | Derived from `texcoord_name`, see above. |
| `set_mode/get_mode`, `is_fixed_function()` | |
| `set_color/get_color`, `set_rgb_scale/get_rgb_scale`, `set_alpha_scale/get_alpha_scale` | Scale values must be 1, 2, or 4. |
| `set_saved_result/get_saved_result` | Publish this stage's output as `CS_last_saved_result` for later stages. |
| `set_tex_view_offset/get_tex_view_offset` | Multiview texture view selection offset. |
| `set_combine_rgb(mode, src0, op0[, src1, op1[, src2, op2]])` / `get_combine_rgb_mode/source{0,1,2}/operand{0,1,2}/num_combine_rgb_operands` | Overload chosen by `mode`'s expected operand count. |
| `set_combine_alpha(...)` / `get_combine_alpha_*` | Same shape, alpha channel. |
| `involves_color_scale/uses_color/uses_primary_color/uses_last_saved_result` | Cached derived flags, see behavior notes. |
| `operator==/!=/<`, `int compare_to(const TextureStage&) const` | Full field-wise comparison. |
| `static TextureStage *get_default()` | Lazily-created shared default stage. |
| `static UpdateSeq get_sort_seq()` | Global stage-sort invalidation counter. |
| `void mark_used_by_auto_shader() const` | Opt this stage into auto-shader rehash notifications. |
| `void write(ostream&[, int indent])`, `void output(ostream&)` | |

## See also

- [InternalName](InternalName.md) — texcoord/tangent/binormal name source
- [TextureStagePool](TextureStagePool.md), [Texture](Texture.md)
- `TextureAttrib`, `ShaderAttrib` (pgraph, see
  [../pgraph/README.md](../pgraph/README.md)) — consumers of `TextureStage`
