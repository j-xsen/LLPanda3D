# StandardMunger

**Source:** `panda/src/display/standardMunger.h` (+ `.I`, `.cxx`)
**Inherits:** StateMunger (external, `panda/src/gobj` — a `GeomMunger` subclass that additionally tracks a `RenderState`; not part of this module) **Inherited by:** backend-specific munger subclasses (e.g. an OpenGL GSG's own munger), outside this module

The default `GeomMunger` that a `GraphicsStateGuardian` creates (typically
from its `make_geom_munger()` override) to adapt application geometry and
render state to what the GSG actually supports before drawing: converting
vertex color/color-scale into vertex data when the GSG can't fake it via
lighting or a texture trick, decomposing unsupported primitive types (tri
fans on hardware without fan support, strip-cut indices, etc.), and
selecting hardware vs. software skinned-vertex animation. Most GSG
subclasses build on `StandardMunger` rather than implementing `GeomMunger`
from scratch.

## Behavior notes

- **The constructor decides once, up front, whether color/color-scale
  munging is needed at all** — it inspects the `RenderState` passed in
  (`ColorAttrib`, `ColorScaleAttrib`, `ShaderAttrib`) against the GSG's
  `get_runtime_color_scale()`/`get_color_scale_via_lighting()`/
  `get_alpha_scale_via_texture()` capabilities. If the GSG can apply color
  directly (`get_runtime_color_scale()`) or a shader/auto-shader is active,
  no munging is scheduled at all (`_munge_color`/`_munge_color_scale` stay
  false) — those cases are handled elsewhere (`glColor4f`-equivalent or the
  shader itself). Munging is only turned on for the specific combination
  the GSG can't fake any other way.
- **`_remove_material` handles a specific interaction with
  color-scale-via-lighting**: if the GSG fakes color scale by enabling
  lighting, and the object has a `MaterialAttrib` but *no* active
  `LightAttrib` lights, that leftover material would incorrectly affect the
  faked lighting — so the munger strips the `MaterialAttrib` from the munged
  state (`munge_state_impl()`) in that case.
- **A documented, accepted bug**: the constructor's comment admits that if
  an object has a `Material` that *would* obscure a `ColorScaleAttrib`'s
  effect under normal (non-munged) rendering, the munger still applies the
  scale via lighting anyway — "it doesn't seem worth the effort to detect
  this contrived situation and handle it correctly."
- **`munge_data_impl()` bolts hardware-skinning detection onto the same pass
  as color munging** — it inspects the vertex data's `GeomVertexAnimationSpec`
  and the GSG's `get_max_vertex_transforms()`/`get_max_vertex_transform_indices()`
  to decide whether to request hardware vertex blending
  (`animation.set_hardware(...)`), preferring an indexed transform palette
  when `matrix-palette` is enabled and it would actually reduce per-vertex
  blend count, falling back to a plain non-indexed table otherwise. This
  runs even when no color munging is needed, since format conversion
  (`convert_to()`) only happens if the resulting format actually differs
  from the original (checked via pointer equality after `munge_format()`).
- **`munge_geom_impl()` and `premunge_geom_impl()` are near-identical** —
  both compare `geom->get_geom_rendering()` against
  `gsg->get_supported_geom_rendering()` and, for any unsupported bits, first
  try `geom->decompose()` (which the comment notes is deliberately
  all-or-nothing: a `Geom` mixing strips and fans decomposes both if fans
  alone are unsupported, since no Panda GSG supports strips without also
  supporting fans), then `geom->rotate()` for shade-model mismatches
  (`SM_flat_last_vertex` vs `SM_flat_first_vertex`), then converts indexed
  geometry to nonindexed via `make_nonindexed()` if indexing itself is
  unsupported. The duplication between the two methods is intentional (they
  're-derive `unsupported_bits` fresh after `decompose()` since decomposing
  can itself introduce indexing) — not a copy-paste bug worth flagging.
- **`compare_to_impl()`/`geom_compare_to_impl()` compare only the munger's
  own decision flags** (`_munge_color`, `_munge_color_scale`, `_shader_skinning`,
  `_auto_shader`, `_remove_material`, and the actual color/scale values),
  falling through to `StateMunger::compare_to_impl()`/`geom_compare_to_impl()`
  for the rest — `geom_compare_to_impl()` deliberately omits `_auto_shader`/
  `_remove_material` from its comparison since those don't affect
  `munge_geom()`'s output, only `munge_state()`'s.

## API

| Signature | Notes |
|---|---|
| `StandardMunger(GraphicsStateGuardianBase *gsg, const RenderState *state, int num_components, NumericType numeric_type, Contents contents)` | The extra three params describe the GSG's preferred vertex color format, reused if color munging is needed. |
| `virtual ~StandardMunger()` | |
| `INLINE GraphicsStateGuardian *get_gsg() const` | Downcasts `GeomMunger::get_gsg()`. |
| `protected: virtual CPT(GeomVertexData) munge_data_impl(const GeomVertexData*)` | Color/color-scale munging + hardware-skinning format conversion. |
| `protected: virtual void munge_geom_impl(CPT(Geom)&, CPT(GeomVertexData)&, Thread*)` | Primitive-type adaptation (decompose/rotate/make_nonindexed) for unsupported `GeomRendering` bits. |
| `protected: virtual void premunge_geom_impl(CPT(Geom)&, CPT(GeomVertexData)&)` | Same logic as `munge_geom_impl()`, run at a different point in the pipeline (pre-munge vs. munge). |
| `protected: virtual int compare_to_impl(const GeomMunger*) const` / `int geom_compare_to_impl(const GeomMunger*) const` | Munger identity/dedup comparisons; see behavior notes. |
| `protected: virtual CPT(RenderState) munge_state_impl(const RenderState*)` | Strips `ColorAttrib`/`ColorScaleAttrib`/`MaterialAttrib` from the state as dictated by the flags computed in the constructor. |

## See also

- [GraphicsStateGuardian.md](GraphicsStateGuardian.md) — `get_geom_munger()`/`make_geom_munger()` are how a `StandardMunger` gets created and cached per `RenderState`.
