# TextureAttrib

**Source:** `panda/src/pgraph/textureAttrib.h` (+ `.I`, `.cxx`)
**Inherits:** RenderAttrib

Indicates the set of `TextureStage`s and their associated `Texture`s
applied to (or removed from) a node — Panda's multitexture mechanism.
`Texture`/`TextureStage`/`SamplerState` are defined in `panda/src/gobj`
(undocumented); this doc covers only the attrib that carries them through
the scene graph.

## Behavior notes

- Like `LightAttrib`, carries independent `on_stages`/`off_stages` sets
  (plus an `off_all_stages` flag) so composing down the graph
  incrementally adds/removes/overrides stages rather than one state fully
  replacing another. `is_off()`/`get_texture()` are the legacy
  single-texture accessors, kept for the common non-multitexture case —
  `is_off()` is really just "`on_stages` is empty," which `has_all_off()`/
  `get_num_off_stages()` describe more precisely under multitexture.
- On-stages are sorted lazily (`check_sorted()`/`sort_on_stages()`,
  triggered by an `UpdateSeq` bumped when any `TextureStage::set_sort()` is
  called — same lazy-resort pattern as `LightAttrib`). Sort order is by
  stage priority descending, then stage sort ascending, then insertion
  order (`_implicit_sort`) — `CompareTextureStagePriorities`.
  `get_on_ff_stage()`/`get_num_on_ff_stages()` give the subset relevant to
  the classic OpenGL fixed-function multitexture pipeline (excludes stages
  like normal maps that only make sense with shaders); `get_ff_tc_index()`
  gives a stable UV-set index per texcoord name for that subset.
  `_render_stages` (all on-stages) vs. `_render_ff_stages` (FF-only) are
  cached parallel lists, invalidated together.
- `unify_texture_stages(stage)` and `replace_texture(tex, new_tex)` support
  bulk remapping (e.g. flattening/converting during model load).
  `filter_to_max(n)` returns a version capped to the `n` highest-priority
  stages, memoized per-`n` in `_filtered` (keyed by `int`, invalidated by
  `_filtered_seq`) — used when a GSG has a hardware texture-unit limit.
- Overrides `lower_attrib_can_override()`, `has_cull_callback()`/
  `cull_callback()` — unlike most `RenderAttrib`s, `TextureAttrib`
  participates directly in the cull traversal (source not fully shown
  here; likely related to view-frustum-relative Texture staging/eviction
  or similar GSG-driven behavior — check `textureAttrib.cxx` for the
  details when this matters).
- Each on-stage can carry its own `SamplerState` override
  (`add_on_stage(stage, tex, sampler, override)`); if none is given,
  `get_on_sampler()` falls back to the `Texture`'s own default sampler.

## API

| Signature | Notes |
|---|---|
| `static CPT(RenderAttrib) make(Texture *tex)` | Single-texture convenience form |
| `static CPT(RenderAttrib) make_off()` / `make_default()` | Disables texturing |
| `bool is_off() const` / `Texture *get_texture() const` | Legacy single-texture accessors |
| `static CPT(RenderAttrib) make()` | Empty multitexture attrib |
| `static CPT(RenderAttrib) make_all_off()` | Turns off all stages |
| `int get_num_on_stages() const` / `TextureStage *get_on_stage(int n) const` | Sorted render order |
| `int get_num_on_ff_stages() const` / `get_on_ff_stage(int n) const` / `get_ff_tc_index(int n) const` | Fixed-function-relevant subset |
| `bool has_on_stage(TextureStage*) const` / `Texture *get_on_texture(TextureStage*) const` | |
| `const SamplerState &get_on_sampler(TextureStage*) const` | Errors if stage absent |
| `int get_on_stage_override(TextureStage*) const` | Asserts if stage absent |
| `int find_on_stage(const TextureStage*) const` | |
| `int get_num_off_stages() const` / `get_off_stage(int n) const` / `has_off_stage(TextureStage*) const` | |
| `bool has_all_off() const` / `is_identity() const` | |
| `CPT(RenderAttrib) add_on_stage(stage, tex, override=0)` / `add_on_stage(stage, tex, sampler, override=0)` | |
| `CPT(RenderAttrib) remove_on_stage(TextureStage*) const` | |
| `CPT(RenderAttrib) add_off_stage(TextureStage*, override=0) const` / `remove_off_stage(TextureStage*) const` | |
| `CPT(RenderAttrib) unify_texture_stages(TextureStage*) const` | Collapses all on-stages onto one stage |
| `CPT(RenderAttrib) replace_texture(Texture *tex, Texture *new_tex) const` | Bulk texture substitution |

## Usage

```cpp
node_path.set_texture(tex);                     // NodePath wrapper for the simple case
node_path.set_texture(stage1, tex1);
node_path.set_texture(stage2, tex2, 1);          // override priority 1
node_path.clear_texture();
```

## See also

[README — the state pipeline](README.md#the-state-pipeline),
[RenderAttrib](RenderAttrib.md), [TexGenAttrib](TexGenAttrib.md),
[TexMatrixAttrib](TexMatrixAttrib.md), [LightAttrib](LightAttrib.md)
