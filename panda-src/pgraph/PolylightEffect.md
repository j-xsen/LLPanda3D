# PolylightEffect

**Source:** `panda/src/pgraph/polylightEffect.h` (+ `.I`, `.cxx`)
**Inherits:** RenderEffect

Attaches a group of [PolylightNode](PolylightNode.md)s ("light group") to a
node so its color is modulated based on proximity to those lights, at cull
time. `PolylightNode` is a cheap, non-realtime (no actual GPU lighting)
approximation intended for effects like flickering torches/night scenes,
computed entirely on the CPU during culling by adjusting a
`ColorScaleAttrib` rather than issuing real light state.

## Behavior notes

- `ContribType`: `CT_proximal` (default) contributes only from lights the
  node is actually within `light_radius` of; `CT_all` divides by the full
  light-group size regardless of range, avoiding a color "snap" at light
  volume boundaries at the cost of dimmer average color.
- `has_cull_callback()` returns `false` when the light group is empty —
  no-op fast path.
- `cull_callback()` calls `do_poly_light()` to compute a `ColorScaleAttrib`
  and composes it into the node's cull-time `RenderState`.
- `do_poly_light()` (the actual algorithm, ~200 lines): for each enabled
  light in range, computes a camera/avatar dot-product-based intensity
  term, applies the light's `AttenuationType` (`ALINEAR` or `AQUADRATIC`
  falloff curve) and flicker state, accumulates weighted RGB, then blends
  in the scene's ambient color scale by `weight` before dividing back out
  the scene color (so the final result isn't double-scaled by a day/night
  cycle that also scales `scene_color`). `effect_center`'s Z component
  toggles whether height is counted in the distance calc (0 = ignore
  height, treat as 2D distance) — a deliberate simplification for
  avatar-height lights.
- Immutable-value pattern like the other effects: `add_light()`/
  `remove_light()`/`set_weight()`/`set_contrib()`/`set_effect_center()`
  all copy-construct and return a new `PolylightEffect`; the private copy
  constructor exists specifically to support this (mutating in place isn't
  possible since instances are shared/interned).
- Debug logging for the whole distance/intensity/color computation is
  gated behind the `polylight-info` config variable (module README).
- Comments in source ("Asad: I don't think we should monkey with the
  weight") indicate the default weight of `1.0` is considered
  load-bearing/tuned — don't casually change `make()`'s default.

## API

| Method | Notes |
|---|---|
| `static CPT(RenderEffect) make()` | Empty light group, weight 1.0, `CT_proximal`, center origin |
| `static CPT(RenderEffect) make(PN_stdfloat weight, ContribType contrib, const LPoint3 &effect_center)` | |
| `static CPT(RenderEffect) make(PN_stdfloat weight, ContribType contrib, const LPoint3 &effect_center, const LightGroup &lights)` | `LightGroup = pvector<NodePath>` |
| `CPT(RenderEffect) add_light(const NodePath &light) const` | Returns new effect |
| `CPT(RenderEffect) remove_light(const NodePath &light) const` | Returns new effect; logs if not found |
| `CPT(RenderEffect) set_weight(PN_stdfloat w) const` | Returns new effect |
| `CPT(RenderEffect) set_contrib(ContribType c) const` | Returns new effect |
| `CPT(RenderEffect) set_effect_center(const LPoint3 &ec) const` | Returns new effect |
| `PN_stdfloat get_weight() const` | |
| `ContribType get_contrib() const` | |
| `LPoint3 get_effect_center() const` | |
| `bool has_light(const NodePath &light) const` | |

`ContribType` enum: `CT_proximal`, `CT_all`.

## Usage

```cpp
NodePath render("render");
NodePath torch_light = render.attach_new_node(new PolylightNode("torch"));
DCAST(PolylightNode, torch_light.node())->set_radius(10.0);

model_np.set_effect(
  PolylightEffect::make()->add_light(torch_light));
```

## See also

- [RenderEffect](RenderEffect.md) — base class
- [PolylightNode](PolylightNode.md) — the light node type this effect reads
- [../pgraph/README.md](README.md) — `polylight-info` config variable
