# pgraphnodes — Concrete Scene Graph Node Types

**Source:** `panda/src/pgraphnodes/`

`pgraphnodes` holds `PandaNode` subclasses that don't belong in `pgraph`
itself (see [../pgraph/README.md](../pgraph/README.md)) or `gobj` (see
[../gobj/README.md](../gobj/README.md)): the concrete `Light`
implementations, LOD/child-selection nodes, a handful of utility nodes, and
two non-node tools (`SceneGraphAnalyzer`, `ShaderGenerator`). `pgraph`
defines the abstract `Light` mixin interface and references these concrete
subclasses without implementing them — this module is where they live.

Excluded from these docs (not real public API surface):
- `config_pgraphnodes.h/.cxx` — folded into **Config variables** below
- `lodNodeType.h` — not a class, just the free-standing `LODNodeType` enum;
  folded into [LODNode.md](LODNode.md)
- `fadeLodNodeData.h` (`FadeLODNodeData`) — small, tightly-coupled
  per-instance cross-fade state; folded into [FadeLODNode.md](FadeLODNode.md)
- `nodeCullCallbackData.h` (`NodeCullCallbackData`) — small,
  tightly-coupled cull-time callback payload; folded into
  [CallbackNode.md](CallbackNode.md)

## Class hierarchy

```
Light (mixin, see ../pgraph/Light.md — defined in pgraph, implemented here)
  LightNode : Light, PandaNode                — non-lens light base (no frustum/direction)
    AmbientLight                              — uniform ambient contribution, no position/direction
  LightLensNode : Light, Camera (../pgraph/Camera.md) — lens-based light base (has a frustum/direction)
    DirectionalLight                          — parallel rays, direction only, no falloff
    PointLight                                — omnidirectional, position + distance falloff
      SphereLight                             — PointLight with a spherical (non-point) light volume
    Spotlight                                 — PointLight-like + a cone (uses its Lens as the cone shape)
    RectangleLight                            — area light emitting from a rectangular surface

PandaNode
  LODNode                                     — distance-based child visibility switching (see LODNodeType below)
    FadeLODNode                               — LODNode + cross-fade transition between LOD levels
  SelectiveChildNode                          — abstract "render exactly one child" base
    SwitchNode                                — explicit index-selected child
    SequenceNode : SelectiveChildNode, AnimInterface (external, chan — undocumented) — time-cycled child
  CallbackNode                                — user draw/cull callback hook
  ComputeNode                                 — dispatches a compute shader (see gobj's Shader/ShaderBuffer)
  UvScrollNode                                — animated texture-coordinate scrolling

TypedReferenceCount
  ShaderGenerator                             — auto-generates a Shader (../gobj/Shader.md) from a RenderState

(standalone, not a scene graph node)
  SceneGraphAnalyzer                          — traversal/stats utility (node/vertex/triangle counts, etc.)
```

## Lights: `LightNode` vs. `LightLensNode`

Every concrete light implements the abstract `Light` interface (defined in
`pgraph`, see [../pgraph/Light.md](../pgraph/Light.md)) alongside a real
`PandaNode`-derived base — a light is attached to the scene graph like any
other node (via `LightAttrib`) so it can be positioned/animated by
reparenting, and so its contribution can be enabled/disabled per-subtree.

Two node bases exist because not every light needs a projection:
`AmbientLight` has no position, direction, or falloff at all, so it only
needs `LightNode` (`PandaNode` + `Light`). Every other light needs a
direction and/or a shaped volume, and reuses `Camera`'s `Lens` machinery to
get it "for free" — `LightLensNode` inherits `Camera` itself, so a
`Spotlight`'s cone shape is literally its `Lens`'s frustum, and shadow
mapping (rendering the scene from the light's point of view) can reuse the
exact same `DisplayRegion`/`Lens`-driven render path as an ordinary camera.

## Config variables (from `config_pgraphnodes.h`/`.cxx`)

| Variable | Purpose |
|---|---|
| `default-lod-type` | Default `LODNodeType` (`LNT_pop` or `LNT_fade`) for newly-created `LODNode`s |
| `support-fade-lod` | Globally enable/disable cross-fade LOD transitions |
| `lod-fade-time` | Duration of a `FadeLODNode` cross-fade, in seconds |
| `lod-fade-bin-name` / `lod-fade-bin-draw-order` | Cull bin (and its draw order) used for the transparent fade-out geometry during a transition |
| `lod-fade-state-override` | `RenderState` override level applied to force transparency during a fade |
| `verify-lods` | Extra validation that LOD switch distances are correctly ordered |
| `parallax-mapping-samples` / `parallax-mapping-scale` | `ShaderGenerator`'s auto-generated parallax-mapping shader quality/strength knobs |

## See also

- [pgraph](../pgraph/README.md) — `Light` interface, `Camera`, `AuxSceneData`,
  the scene graph `PandaNode` these all extend
- [gobj](../gobj/README.md) — `Shader`/`ShaderBuffer` (`ComputeNode`,
  `ShaderGenerator`), `Material` (`ShaderGenerator`'s auto-shader input)
