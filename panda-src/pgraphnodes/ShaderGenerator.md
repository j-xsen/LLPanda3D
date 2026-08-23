# ShaderGenerator

**Source:** `panda/src/pgraphnodes/shaderGenerator.h` (+ `.I`, `.cxx`, ~2.5k
lines — the biggest class in `pgraphnodes`)
**Inherits:** `TypedReferenceCount`
**Inherited by:** (none)

Panda's "auto shader" / "next-gen fixed function pipeline" — given a
`RenderState` (lights, `MaterialAttrib`, `TextureAttrib` stages, fog,
transparency, color scale, light ramps, …), it synthesizes Cg shader source
implementing the requested lighting/texturing pipeline, instead of
requiring a hand-written `Shader`. Enabled per-`RenderState` via
`ShaderAttrib::auto_shader()`); one `ShaderGenerator` instance exists per
`GraphicsStateGuardian`.

The header declares this class **twice**: once under `#ifdef HAVE_CG` (the
real implementation documented below) and once under `#else` — a
do-nothing stub with the same public signature, used when Panda is built
without Cg support, whose `synthesize_shader()` presumably returns nothing
useful. This doc covers the real, Cg-enabled version.

## Behavior notes

- **Results are cached per unique `ShaderKey`, not per `RenderState`.**
  `synthesize_shader()` first calls `analyze_renderstate()` to reduce the
  full `RenderState` down to a `ShaderKey` — a compact struct capturing
  only what actually affects the generated shader's *structure* (texture
  stage modes/combine ops, which maps are present — normal/height/glow/
  gloss/emission, light types and their flags, fog mode, alpha test mode,
  material flag bits, color type, animation spec, …), explicitly **not**
  the actual numeric values (a material's diffuse color, a light's
  attenuation, a fog's density). Two different `RenderState`s that only
  differ in those excluded numeric values hash to the same `ShaderKey` and
  reuse the identical generated shader — the numeric values themselves are
  bound as separate shader inputs at draw time, not baked into the source.
  A `phash_map<ShaderKey, CPT(ShaderAttrib)>` (`_generated_shaders`) holds
  the cache; a cache hit skips shader synthesis entirely (`lookup_collector`
  vs. `synthesize_collector` PStat timers separate the two paths).
- **`analyze_renderstate()` is also where the `Material` "auto shader"
  flag gets set** — it calls `mat->mark_used_by_auto_shader()` on the
  attached material (if any) whenever a `RenderState` with
  `auto_shader()` enabled is analyzed. This is the call site behind the
  bug documented in [../gobj/Material.md](../gobj/Material.md):
  `mark_used_by_auto_shader()` sets `Material::F_used_by_auto_shader`
  (`0x800`), but `Material::is_used_by_auto_shader()` actually checks the
  unrelated `F_attrib_lock` bit (`0x040`) — so the "should changing this
  material trigger `rehash_generated_shaders()`" check that
  `is_used_by_auto_shader()` is meant to gate never actually fires from
  this path. Not re-documented in depth here; see `Material.md` for the
  full analysis.
- **`rehash_generated_shaders()`'s effectiveness depends on
  `uniquify-states`.** With state interning on (the `RenderState`
  registry, see [../pgraph/README.md](../pgraph/README.md#the-state-pipeline)),
  it can walk every live, interned `RenderState` directly, re-derive each
  one's `ShaderKey`, and patch `state->_generated_shader` in place if a
  different (or no) cached shader now applies. Without interning, there's
  no single registry of all live states to walk, so it instead bumps a
  *global* sequence number via
  `GraphicsStateGuardianBase::mark_rehash_generated_shaders()`, forcing a
  coarser, deferred re-check the next time each state is actually used
  for rendering. `clear_generated_shaders()` is the blunter version:
  drops the entire `_generated_shaders` cache and every state's cached
  pointer unconditionally (but does **not** clear the GSG's compiled-Cg-
  program cache — a subsequent `synthesize_shader()` call for a
  previously-seen `ShaderKey` still has to regenerate the Cg *source*, but
  compiling that source may still hit the driver/GSG's own shader-object
  cache).
- **Register allocation is manual and profile-dependent.** `alloc_vreg()`/
  `alloc_freg()` hand out vertex/fragment shader input registers from a
  small fixed pool (`ATTR1`/`ATTR3`-`ATTR15` or `TEXCOORD0`-`TEXCOORD15`
  depending on `_use_generic_attr`, itself decided from GSG capability
  bits — HLSL/Direct3D 9 targets don't get the `ATTR*` generic-attribute
  names). Running out of registers (more texture stages / tangent-space
  needs than fit) isn't guarded against in the header; check the `.cxx`
  synthesis logic directly if you hit register-related shader compile
  failures on an unusually complex `RenderState`.
- The class-level doc comment in the header lists the supported feature
  set explicitly: flat/vertex colors, lighting, (multiple) normal maps, a
  single gloss map, a single glow map, materials (but not runtime updates
  to material *values* — only the flags feed the `ShaderKey`, per above),
  2D/1D/3D/cube/array textures, all texture combine modes, color scale,
  light ramps (cartoon shading), shadow mapping, most texgen modes,
  texture matrices, linear/exp/exp2 fog, and vertex animation. Anything
  outside that list requires a hand-written `Shader`.

## API

| Method | Notes |
|---|---|
| `ShaderGenerator(const GraphicsStateGuardianBase *gsg)` | Normal constructor — derives register-allocation strategy and shadow-filter support from the GSG's capabilities. |
| `ShaderGenerator(const Shader::ShaderCaps &caps, bool use_shadow_filter)` | Lower-level constructor taking capability flags directly. |
| `synthesize_shader(const RenderState *rs, const GeomVertexAnimationSpec &anim)` | Main entry point: returns a cached or newly-generated `ShaderAttrib` implementing `rs`'s requested pipeline for vertices animated per `anim`. |
| `rehash_generated_shaders()` | Re-validate/patch already-generated shaders in place after external state affecting shader structure changed (e.g. a light or material flag) — see Behavior notes for the `uniquify-states` caveat. |
| `clear_generated_shaders()` | Unconditionally drop the entire generated-shader cache, forcing full regeneration on next use. |

## Usage

`ShaderGenerator` is invoked automatically by the render pipeline once
`ShaderAttrib::auto_shader()` is set on a `RenderState` — application code
rarely calls `synthesize_shader()` directly:

```cpp
NodePath np = ...;
np.set_shader_auto();  // enables ShaderAttrib::auto_shader() on this subtree
```

## See also

- [../gobj/Shader.md](../gobj/Shader.md) — the `ShaderAttrib`/`Shader`
  objects this class produces
- [../gobj/Material.md](../gobj/Material.md) — the
  `is_used_by_auto_shader()`/`mark_used_by_auto_shader()` bug this class's
  `analyze_renderstate()` triggers
- [../pgraph/README.md](../pgraph/README.md#the-state-pipeline) —
  `RenderState` interning, which `rehash_generated_shaders()`'s fast path
  depends on
