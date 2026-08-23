# pgraph — Scene Graph Core

**Source:** `panda/src/pgraph/`

`pgraph` is the heart of Panda3D's C++ engine: the scene graph itself
(`PandaNode`, `NodePath`), the immutable render-state pipeline that every
node carries (`RenderState`/`RenderAttrib`/`RenderEffect`/`TransformState`),
the cull traversal that turns a scene graph + camera into a list of things
to draw, and the model loading/caching machinery (`Loader`, `BamFile`,
`ModelPool`). Every other rendering-adjacent module (`display`, `gobj`,
`pgraphnodes`, `egg2pg`, …) either builds nodes that plug into this graph or
consumes state objects defined here.

Excluded from these docs (not real public API surface):
- `*_ext.h/.cxx` — Python scripting glue (`bamFile_ext`, `nodePath_ext`,
  `pandaNode_ext`, `renderState_ext`, `shaderAttrib_ext`,
  `shaderInput_ext`, `transformState_ext`, `nodePathCollection_ext`,
  `loaderFileTypeRegistry_ext`)
- `pythonLoaderFileType.h/.cxx` — Python-only `LoaderFileType` glue
- `test_pgraph.cxx` — standalone test program
- `p3pgraph_composite*.cxx` — build-system amalgamation files
- `camera.N` — interrogate directive file, not code
- `config_pgraph.h/.cxx` — folded into **Config variables** below
- `cullBinEnums.h` — folded into [CullBinManager](CullBinManager.md) (the
  enum it declares, `BinType`)

## Class hierarchy

```
TypedWritableReferenceCount, Namable, LinkedListNode
  PandaNode                          — one node in the scene graph
    LensNode                         — carries a Lens (see mathutil, undocumented)
      Camera                         — a viewpoint; DisplayRegions render through one
    Fog                              — linear/exponential fog falloff, attached via FogAttrib
    GeomNode                         — leaf node holding renderable Geoms (see gobj, undocumented)
    ModelNode                        — marks the root of a loaded model subtree
      ModelRoot                      — ModelNode specifically for Loader-loaded files
    OccluderNode                     — a portal-culling occluder polygon
    PlaneNode                        — holds a clipping/reflection plane
    PolylightNode                    — a non-realtime "polylight" light source
    PortalNode                       — a portal-culling portal polygon

NodePath                             — a value-type handle to a PandaNode chain (see below)
  NodePathComponent                  — one link in a NodePath's parent chain (ref-counted, shared)
  NodePathCollection                 — ordered list of NodePaths
  WeakNodePath                       — non-owning NodePath reference
  WorkingNodePath                    — internal traversal-time NodePath stand-in

RenderState / RenderAttrib / RenderEffect / RenderEffects / TransformState
  — see "The state pipeline" below; RenderAttrib has ~25 direct subclasses
    (AlphaTestAttrib, ColorAttrib, CullFaceAttrib, DepthTestAttrib,
    FogAttrib, LightAttrib, MaterialAttrib, ShaderAttrib, TextureAttrib,
    TransparencyAttrib, …) and RenderEffect has ~8 (BillboardEffect,
    CompassEffect, DecalEffect, ScissorEffect, ShowBoundsEffect, …)

Light                                — mixin interface for light-casting nodes;
                                        concrete lights (PointLight, Spotlight, …)
                                        live in panda/src/pgraphnodes (undocumented)

CullTraverser / CullTraverserData / CullHandler / CullResult / CullBin /
CullBinManager / CullPlanes / CullableObject / SceneSetup
  — see "Cull pipeline" below

Loader / BamFile / LoaderFileType / LoaderFileTypeBam /
LoaderFileTypeRegistry / ModelPool / ModelLoadRequest / ModelSaveRequest /
ModelFlattenRequest / GeomTransformer / SceneGraphReducer / CacheStats
  — see "Loading and model management" below
```

## NodePath vs. PandaNode

`PandaNode` is the actual scene-graph node — reference-counted, and (unlike
most Panda3D objects) **allowed to have multiple parents**, since the same
node can be instanced into the graph at more than one place. Because a node
alone doesn't identify *which* instance/path you mean, application code
almost never touches `PandaNode` directly.

`NodePath` is a lightweight value-type handle to one specific path from a
root down to a node — it's what application code actually holds and passes
around (copied by value, compared by value, storable in STL containers). It
resolves to a chain of `NodePathComponent` objects, each holding one
`(PandaNode*, parent NodePathComponent*)` link; components are shared and
ref-counted so that many `NodePath`s through the same graph region reuse
the same chain. `WeakNodePath` holds a non-owning reference (useful for
caches that shouldn't keep a subtree alive); `NodePathCollection` is an
ordered/searchable list of `NodePath`s, as returned by
`NodePath::find_all_matches()` etc. `WorkingNodePath` is an internal,
allocation-light stand-in used during traversal/search — not something
application code constructs.

## The state pipeline

Every `PandaNode` carries two immutable, ref-counted, **automatically
interned** objects: a `RenderState` (the node's own local rendering state)
and `RenderEffects` (behaviors like billboarding that affect how the node
itself is handled, as opposed to how it's drawn). A `RenderState` is a set
of at most one `RenderAttrib` per `TypeHandle` (e.g. one `ColorAttrib`, one
`TextureAttrib`, …); `RenderEffects` is a similar set of `RenderEffect`s.

**Interning:** `RenderState::make()` (and the equivalent for
`RenderEffects`/`TransformState`) never returns a fresh object for
identical content — it looks up a global registry keyed by content hash and
returns the existing shared instance, incrementing its refcount. This makes
state comparison an O(1) pointer-equality check almost everywhere in the
renderer, and is why states are always handled through `CPT()`/`PT()`
smart pointers, never copied or mutated in place — "mutating" a state means
calling `add_attrib()`/`remove_attrib()`, which composes a *new* interned
state and returns it. `garbage_collect_states`/`state_cache`/
`uniquify_states`/`transform_cache`/`uniquify_transforms` (config vars
below) tune this caching. `AccumulatedAttribs` is the mutable accumulator
used internally (e.g. by `SceneGraphReducer`) to fold a chain of states
into one before flattening; `AttribNodeRegistry` and `RenderAttribRegistry`
are separate lookup-by-name/type registries used by tools, not part of the
hot rendering path. `StateMunger` (a `GeomMunger` subclass) converts a
`RenderState` into GSG-specific vertex-format munging decisions.

`TransformState` follows the same interned-immutable pattern but is not a
`RenderAttrib` — it's tracked on `PandaNode` as a separate slot (position/
rotation/scale, or an arbitrary matrix) since it needs cheaper composition
(matrix multiply) than the general attribute-override mechanism.

## Cull pipeline

Once per frame, `SceneSetup` captures a `Camera`'s viewpoint into a fixed
snapshot, and a `CullTraverser` walks the scene graph from a root
`NodePath`, accumulating `RenderState` down each branch and testing bounding
volumes against the view frustum (plus any active `CullPlanes` — clip
planes, portal-culled polygon clips, occluder clips).  `CullTraverserData`
is the per-node traversal state passed down the recursion.  Visible
renderable objects are wrapped as `CullableObject`s and handed to a
`CullHandler` (`CullResult` is the standard implementation), which sorts
them into `CullBin`s (opaque, transparent, fixed, unsorted, …) as tracked
by the global `CullBinManager`. `PortalNode`/`PortalClipper` implement
portal-based visibility culling (clipping the frustum through portal
polygons room-to-room); `OccluderNode` implements occluder-volume culling.
`GeomDrawCallbackData` is the `CallbackData` passed to a user draw callback
attached via a `CallbackNode` (see `pgraphnodes`, undocumented).

## Loading and model management

`Loader` is the front-end for loading model files asynchronously (via
`ModelLoadRequest`/`ModelSaveRequest`/`ModelFlattenRequest`, `AsyncTask`
subclasses run on a task chain) or synchronously; it dispatches to a
registered `LoaderFileType` (looked up by extension through
`LoaderFileTypeRegistry`) — `LoaderFileTypeBam` is the built-in handler for
Panda's native `.bam`/`.pz` format, implemented via `BamFile`.  Loaded
models are wrapped with a `ModelRoot` (a `ModelNode` subclass) at their
root and cached by `ModelPool` so repeat loads of the same path return a
shared copy.  `GeomTransformer` and `SceneGraphReducer` implement geometry
flattening (combining sibling Geoms/states to reduce node/state count —
`NodePath::flatten_strong()` etc. delegate to these).  `CacheStats` tracks
hit/miss counters for the state/transform caches described above.

## Config variables (from `config_pgraph.h`/`.cxx`)

| Variable | Purpose |
|---|---|
| `fake-view-frustum-cull` | Disable actual frustum culling but still compute what would've been culled, for profiling |
| `clip-plane-cull` | Enable/disable clip-plane-based culling |
| `allow-portal-cull` / `debug-portal-cull` | Enable portal culling / verbose portal debug output |
| `show-occluder-volumes` | Render occluder volumes visibly, for debugging |
| `unambiguous-graph` | Detect/report ambiguous NodePath references (multi-parent) |
| `detect-graph-cycles` | Runtime cycle detection when reparenting |
| `no-unsupported-copy` | Disallow copying certain graph structures the engine can't safely duplicate |
| `allow-unrelated-wrt` | Allow `wrt()` transforms between NodePaths with no common ancestor (falls back to render root) |
| `paranoid-compose` / `paranoid-const` | Extra validation when composing states / on const state access |
| `compose-componentwise` | TransformState composition strategy |
| `auto-break-cycles` | Automatically break reference cycles in the scene graph |
| `garbage-collect-states` / `garbage-collect-states-rate` | Periodic sweep of unreferenced interned RenderStates |
| `transform-cache` / `state-cache` | Enable the TransformState/RenderState interning caches |
| `uniquify-transforms` / `uniquify-states` / `uniquify-attribs` | Enable interning at each pipeline level |
| `retransform-sprites` | Recompute sprite billboard transforms every frame vs. caching |
| `depth-offset-decals` | Implement DecalEffect via depth offset instead of polygon offset |
| `max-collect-vertices` / `max-collect-indices` | Caps for GeomTransformer/SceneGraphReducer flattening |
| `premunge-data` | Pre-munge vertex data for the GSG at load time |
| `preserve-geom-nodes` | Don't flatten separate GeomNodes together |
| `flatten-geoms` | Flatten individual Geoms within a node during load |
| `max-lenses` | Max Lens objects a single Camera/LensNode can hold |
| `polylight-info` | Debug logging for PolylightNode/PolylightEffect |
| `show-vertex-animation` / `show-transparency` | Debug-tint rendering of animated/transparent geometry |
| `m-dual` / `m-dual-opaque` / `m-dual-transparent` / `m-dual-flash` | Dual (front+back) transparency rendering mode tuning |
| `load-file-type` | Preload specific `LoaderFileType` plugins by name |
| `default-model-extension` | Extension assumed when a load path has none |
| `allow-live-flatten` | Allow flattening a scene graph currently attached to render |
| `filled-wireframe-apply-shader` | Apply the node's shader when rendering RenderModeAttrib's filled-wireframe mode |

## See also

- [pgui](../pgui/README.md), [text](../text/README.md), [event](../event/README.md),
  [framework](../framework/README.md), [display](../display/README.md) — other
  documented modules; `display`'s `GraphicsStateGuardian` is the consumer
  of the `RenderState`/`CullResult` output produced here.
