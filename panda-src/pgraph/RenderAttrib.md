# RenderAttrib

**Source:** `panda/src/pgraph/renderAttrib.h` (+ `.I`, `.cxx`)
**Inherits:** TypedWritableReferenceCount
**Inherited by:** ~25 concrete subclasses — AlphaTestAttrib, AntialiasAttrib,
AudioVolumeAttrib, AuxBitplaneAttrib, ClipPlaneAttrib, ColorAttrib,
ColorBlendAttrib, ColorScaleAttrib, ColorWriteAttrib, CullBinAttrib,
CullFaceAttrib, DepthOffsetAttrib, DepthTestAttrib, DepthWriteAttrib,
FogAttrib, LightAttrib, LightRampAttrib, LogicOpAttrib, MaterialAttrib,
RenderModeAttrib, RescaleNormalAttrib, ScissorAttrib, ShadeModelAttrib,
ShaderAttrib, StencilAttrib, TexGenAttrib, TexMatrixAttrib, TextureAttrib,
TransparencyAttrib

Abstract base for every "attrib" that can be added to a [RenderState](RenderState.md)
to control the appearance of geometry (as opposed to [RenderEffect](RenderEffect.md),
which controls how the *node itself* is handled — billboarding, decaling —
rather than propagating to leaves). A RenderAttrib has the same effect
whether it's assigned at a leaf or several nodes above; `RenderState`
composes attribs of the same type down the graph via `compose()`. Never
constructed directly by application code — each subclass exposes its own
`make()` factory.

## Behavior notes

- **Interning**, same pattern as `RenderState`: subclass `make()` methods
  build a raw instance and pass it through `return_new()`, which either
  returns it as-is (`uniquify_attribs` off) or, via `return_unique()`, looks
  it up in a global content-hash table (`compare_to_impl()`+`get_hash_impl()`
  define equality/hash) and returns the existing shared instance —
  disabled globally by the `state-cache` config variable. `calc_hash()`
  mixes the subclass's `get_hash_impl()` with the concrete `TypeHandle`, so
  two different subclasses never collide even with identical hash impls.
- **Default virtual behavior a subclass overrides selectively:**
  `compose_impl(other)`/`invert_compose_impl(other)` default to just
  returning `other` unchanged — i.e. "a later attrib of the same type fully
  replaces an earlier one" is the default composition rule. Attribs that
  need genuine combination (e.g. `ColorScaleAttrib` multiplying scale
  factors, `TexMatrixAttrib` composing matrices) override these.
  `compare_to_impl()`/`get_hash_impl()` default to `0` (all instances equal)
  — a subclass with any real per-instance data **must** override both or
  every instance will collapse to one interned object.
  `lower_attrib_can_override()` defaults to `false`; see
  [RenderState](RenderState.md)'s per-slot composition rule for what setting
  it `true` does (lets a lower-override-priority instance still win over a
  higher one during `RenderState::compose()`).
- `compare_to(other)` (the public, non-`_impl` version) first compares
  `TypeHandle`s and only calls `compare_to_impl()` when both sides are the
  same concrete subclass — this is why `compare_to_impl` never needs to
  type-check its argument.
- `has_cull_callback()`/`cull_callback()` default to false/no-op; a handful
  of attribs (mainly ones needing GSG-side setup, like clip planes) override
  them, and `RenderState::has_cull_callback()`/`cull_callback()` OR
  together every attrib's answer.
- `register_slot(type_handle, sort, default_attrib)` is how each concrete
  subclass claims its slot index at static-init time — see
  [RenderAttribRegistry](RenderAttribRegistry.md).
- `garbage_collect()` is gated by the *same* `garbage_collect_states` /
  `garbage_collect_states_rate` config variables as `RenderState`'s
  collector, even though the variable name says "states" — it's a shared
  incremental sweep budget.

## API

| Method | Notes |
|---|---|
| `CPT(RenderAttrib) compose(other) const` | Combine with a same-type attrib below this one; default is "other wins" |
| `CPT(RenderAttrib) invert_compose(other) const` | Compose with this attrib's inverse; used for relative-state computation |
| `virtual bool lower_attrib_can_override() const` | Opt-in override-priority escape hatch (see notes) |
| `int compare_to(other) const` | Type-then-content ordering, used by the interning table |
| `size_t get_hash() const` | Cached content+type hash |
| `CPT(RenderAttrib) get_unique() const` | Re-resolve to the canonical interned pointer |
| `virtual int get_slot() const = 0` | This attrib's registered slot index (every concrete subclass implements) |
| `static int get_num_attribs()` / `list_attribs(ostream&)` | Global interning-table introspection |
| `static int garbage_collect()` | Sweep unreferenced interned attribs now |
| `void output(ostream&) const` / `write(ostream&, int) const` | Debug printing (subclasses override `output`) |

Protected virtuals a subclass implements: `compare_to_impl(other)`,
`get_hash_impl()`, `compose_impl(other)`, `invert_compose_impl(other)`.

## Usage

```cpp
// Subclasses expose make(); RenderAttrib itself is never constructed:
CPT(RenderAttrib) ca = ColorAttrib::make_flat(LColor(1, 1, 1, 1));
CPT(RenderState) s = RenderState::make(ca);
```

## See also

[RenderState](RenderState.md), [RenderAttribRegistry](RenderAttribRegistry.md),
[RenderEffect](RenderEffect.md), module [README](README.md) ("The state pipeline").
