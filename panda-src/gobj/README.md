# gobj — Graphics Objects (Geometry, Textures, Shaders, Lenses)

**Source:** `panda/src/gobj/`

`gobj` is the low-level graphics-object layer that `pgraph` (see
[../pgraph/README.md](../pgraph/README.md)) builds scenes out of and
`display`'s `GraphicsStateGuardian` (see
[../display/README.md](../display/README.md)) renders. It owns: renderable
geometry (`Geom`/`GeomPrimitive`/`GeomVertexData` and the columnar vertex
format system), `Texture` and everything around loading/pooling/streaming
it, `Shader`/`Material`/`SamplerState`, camera projection math (`Lens` and
its subclasses), the CPU-side vertex skinning/animation-blend classes, and
the GSG-agnostic resource-management plumbing (`PreparedGraphicsObjects`,
per-object `*Context` handles, the `SimpleLru`/`AdaptiveLru`/`SimpleAllocator`
caching/paging machinery). A `GraphicsStateGuardian` subclass (e.g. the GL
backend) is what actually turns a `Texture`/`Geom`/`Shader` into GPU calls;
`gobj` defines the API-agnostic data and the "prepared object" handshake, not
the backend-specific upload code itself.

Excluded from these docs (not real public API surface):
- `*_ext.h/.cxx` — Python scripting glue (`texture_ext`,
  `textureCollection_ext`, `geomVertexArrayData_ext`, `internalName_ext`)
- `config_gobj.h/.cxx` — folded into **Config variables** below
- `geomEnums.h` — folded into **Shared enums (GeomEnums)** below (pure
  scoping class, no state — see that section)

## Class hierarchy

```
TypedWritableReferenceCount
  Texture                             — see Texture.md; VideoTexture (Texture + AnimInterface) subclasses it
    VideoTexture                      — texture whose content is decoded from a video stream per-frame
  TextureStage                        — a texture's role/blend-mode slot on a GeomNode/RenderState
  Shader                              — a compiled shader program (Cg/GLSL/HLSL/SPIR-V)
  ShaderBuffer (+ GeomEnums)          — a raw GPU buffer bound as a shader storage/uniform buffer
  Material (+ Namable)                — lighting material (ambient/diffuse/specular/emission/shininess)
  Lens                                — abstract camera projection; see "Lens hierarchy" below
    MatrixLens / OrthographicLens / PerspectiveLens
  InternalName (final)                — interned string identifier (vertex column names, shader inputs, …)
  GeomVertexArrayFormat (final, + GeomEnums)
  GeomVertexFormat (final, + GeomEnums) — ordered set of GeomVertexArrayFormats, itself interned
  SliderTable                         — named VertexSliders for morph/blend-shape targets
  TransformTable                      — named VertexTransforms for hardware skinning palettes
  VertexTransform                     — abstract "produces a matrix" node; VertexSlider — abstract "produces a float" node
    UserVertexTransform / UserVertexSlider — app-supplied constant value

CopyOnWriteObject (+ GeomEnums, copy-on-write handle pattern)
  Geom                                — one drawable: a GeomVertexData + one or more GeomPrimitives
  GeomPrimitive                       — abstract indexed-topology primitive; see "GeomPrimitive subclasses" below
  GeomVertexData                      — a set of named, interleaved-or-not vertex arrays
  GeomVertexArrayData (+ SimpleLruPage) — one physical array (one GeomVertexArrayFormat's worth of rows)
  TransformBlendTable                 — per-vertex list-of-(VertexTransform, weight) for CPU soft-skinning

GeomEnums (scoping only, see below)
  GeomVertexColumn                    — one named field within an array format (name/type/offset/…)
  GeomVertexReader / GeomVertexWriter — row-at-a-time typed accessors into a GeomVertexArrayData
    GeomVertexRewriter                — inherits both Reader and Writer (read-modify-write in place)
  GeomVertexAnimationSpec             — describes a format's vertex-animation mode (none/panda/hardware)
  GeomMunger (+ TypedReferenceCount)  — converts a GeomVertexData into a GSG-specific-friendly layout
  ShaderBuffer                        — (listed above; also inherits GeomEnums)

TypedReferenceCount
  QueryContext                        — GPU async query handle
    OcclusionQueryContext / TimerQueryContext

TypedObject
  SavedContext                        — base of every "GSG-side handle to an uploaded resource" class
    BufferContext (+ LinkedListNode, private)  — buffer-family context base (adds AdaptiveLruPage-tracked residency)
      IndexBufferContext / VertexBufferContext / TextureContext  (each also : AdaptiveLruPage)
    GeomContext                       — GSG-side handle for display-list-style cached Geom rendering
    SamplerContext (+ SimpleLruPage)  — GSG-side handle for an uploaded sampler object
    ShaderContext                     — GSG-side compiled/linked shader program handle

ReferenceCount
  PreparedGraphicsObjects             — per-GSG registry owning all of the above *Context objects
  GeomCacheEntry                      — one entry in the global Geom-munging result cache
  TexturePeeker                       — CPU-side random-access reader into a Texture's image data

SimpleLruPage / SimpleLru / AdaptiveLru / AdaptiveLruPage / SimpleAllocator / SimpleAllocatorBlock
  — see "Residency tracking: LRUs and allocators" below
  VertexDataPage (: SimpleAllocator, SimpleLruPage)
  VertexDataSaveFile (: SimpleAllocator)
  VertexDataBlock (: SimpleAllocatorBlock, ReferenceCount)
  VertexDataBook, VertexDataBuffer     — non-inheriting helpers around the above

AsyncTask (see event/README.md)
  AnimateVerticesRequest / TextureReloadRequest — background-thread task wrappers

Not part of any hierarchy above (standalone/static utility classes):
  TexturePool / TextureStagePool / MaterialPool — static name→object caches
  TextureCollection, TexturePoolFilter (: TypedObject), GeomCacheManager,
  SamplerState, TransformBlend, BufferResidencyTracker, BufferContextChain,
  ParamTextureSampler / ParamTextureImage (: ParamValueBase, see putil,
  undocumented)
```

## Shared enums (`GeomEnums`)

`GeomEnums` (`geomEnums.h`) declares no data or behavior — it exists purely
to give a shared scope for enums used across `Geom`, `GeomPrimitive`,
`GeomVertexData`, `GeomVertexFormat`, `GeomVertexColumn`,
`GeomVertexReader`/`Writer`, `GeomMunger`, and `ShaderBuffer` (all of which
inherit it). Every class doc below refers back here instead of
re-explaining these:

| Enum | Values | Meaning |
|---|---|---|
| `UsageHint` | `UH_client`, `UH_stream`, `UH_dynamic`, `UH_static`, `UH_unspecified` | How often vertex/index data changes, ordered most→least dynamic; lets the GSG pick upload strategy (e.g. `UH_client` never uploads to the GPU at all). |
| `NumericType` | `NT_uint8/16/32`, `NT_packed_dcba`, `NT_packed_dabc`, `NT_float32/64`, `NT_stdfloat`, `NT_int8/16/32`, `NT_packed_ufloat` | Physical wire format of one column's values. `NT_stdfloat` resolves to float or double per the `vertices-float64` config var. |
| `Contents` | `C_other`, `C_point`, `C_clip_point`, `C_vector`, `C_texcoord`, `C_color`, `C_index`, `C_morph_delta`, `C_matrix`, `C_normal` | Semantic meaning of a column — drives automatic transforms (e.g. points vs. vectors transform differently) and shader input binding by convention. |
| `PrimitiveType` | `PT_none`, `PT_polygons`, `PT_lines`, `PT_points`, `PT_patches` | Coarse topology family a `GeomPrimitive` subclass belongs to; used for antialiasing mode selection. |
| `ShadeModel` | `SM_uniform`, `SM_smooth`, `SM_flat_first_vertex`, `SM_flat_last_vertex` | Whether per-vertex color/normal data is truly per-vertex or flat-shaded from one representative vertex per face. |
| `GeomRendering` (bitmask) | `GR_indexed_point`, `GR_point*` (size/perspective/aspect/scale/rotate/sprite flags), `GR_triangle_strip`, `GR_triangle_fan`, `GR_line_strip`, `GR_strip_cut_index`, `GR_flat_first/last_vertex`, `GR_render_mode_wireframe/point`, `GR_adjacency`, … | Bit-per-requirement summary of what a `Geom`/`GeomPrimitive` needs the GSG to support; compared against the GSG's own capability bits to decide if munging (via `GeomMunger`) is required. |
| `AnimationType` | `AT_none`, `AT_panda`, `AT_hardware` | Whether a `GeomVertexFormat` carries no vertex animation, CPU-computed (Panda) animation, or GPU hardware-skinned animation — see `GeomVertexAnimationSpec`. |

## Lens hierarchy

`Lens` is the abstract base for camera projection math — it converts
between 3-D view-space points and 2-D film/screen-space coordinates and
back, and exposes the film size/aspect ratio/near-far clip/FOV/keystone/
IOD (stereo interocular distance) parameters that drive that conversion.
`PerspectiveLens` is the common case (field-of-view projection);
`OrthographicLens` has no perspective foreshortening; `MatrixLens` instead
takes an arbitrary user-supplied projection matrix, bypassing the
FOV/near/far parameterization entirely. A `Lens` is not itself a scene
graph node — `LensNode`/`Camera` (`pgraph`, see
[../pgraph/README.md](../pgraph/README.md)) hold one or more `Lens`
objects and give them a position/orientation in the graph.

## `GeomPrimitive` subclasses

`GeomPrimitive` is abstract; concrete subclasses differ only in topology
and how they interpret their index array — `GeomPoints`, `GeomLines`,
`GeomLinestrips`, `GeomTriangles`, `GeomTristrips`, `GeomTrifans`,
`GeomPatches` (arbitrary-size patches for tessellation shaders), and the
`*Adjacency` variants (`GeomLinesAdjacency`, `GeomLinestripsAdjacency`,
`GeomTrianglesAdjacency`, `GeomTristripsAdjacency`) which additionally
carry adjacent-primitive vertices for geometry-shader use. All of them are
documented together in [GeomPrimitive.md](GeomPrimitive.md) rather than
one file each, since each subclass adds only a handful of topology-specific
overrides on top of the shared `GeomPrimitive` machinery.

## Copy-on-write and interning

`Geom`, `GeomPrimitive`, `GeomVertexData`, `GeomVertexArrayData`, and
`TransformBlendTable` all inherit `CopyOnWriteObject`: they're normally
accessed read-only through a `CPT()` (const pointer), and a
`PT(...)::write()`-style call only actually duplicates the underlying data
if it's shared (refcount > 1), otherwise mutates in place. This is the same
pattern `pgraph`'s `RenderState` uses for cheap sharing, but COW rather than
full interning — two `Geom`s with identical vertex data are *not*
automatically merged into one object the way two identical `RenderState`s
are. `GeomVertexFormat` and `GeomVertexArrayFormat`, by contrast, *are*
interned (`GeomVertexFormat::register_format()`), same as `RenderState` —
identical formats collapse to one shared, `final` (non-subclassable)
instance, so format comparison is pointer-equality.

## Residency tracking: LRUs and allocators

Two related but distinct caching subsystems live here:

- **GPU residency (`SimpleLru`/`AdaptiveLru`):** `PreparedGraphicsObjects`
  tracks every uploaded `TextureContext`/`VertexBufferContext`/
  `IndexBufferContext` on a per-GSG `AdaptiveLru` (an LRU that weights
  eviction by both recency and object size/cost, tuned by
  `adaptive-lru-weight`). `BufferResidencyTracker` and
  `BufferContextChain` are supporting bookkeeping for that same system —
  see [PreparedGraphicsObjects.md](PreparedGraphicsObjects.md).
- **Vertex data disk paging (`SimpleAllocator`/`VertexDataPage`):** large
  `GeomVertexArrayData` buffers can be paged out to disk
  (`VertexDataSaveFile`, directory controlled by
  `vertex-save-file-directory`) under memory pressure, tracked through
  `VertexDataPage`/`VertexDataBook`/`VertexDataBlock`, independently of
  GPU residency — a page can be resident in system RAM, paged to disk, or
  both. `SimpleAllocator`/`SimpleAllocatorBlock` is the generic
  best-fit/first-fit block allocator underlying both `VertexDataPage` and
  `VertexDataSaveFile`.

Both subsystems are documented together in
[SimpleLru.md](SimpleLru.md)/[AdaptiveLru.md](AdaptiveLru.md)/
[SimpleAllocator.md](SimpleAllocator.md) plus the `VertexData*` docs.

## `PreparedGraphicsObjects` / `*Context` handshake

A `Texture`, `Geom`'s vertex/index arrays, or a `Shader` doesn't talk to
the GPU API directly. Instead, the GSG calls e.g.
`Texture::prepare_now(PreparedGraphicsObjects *)`, which returns (creating
if needed) a `TextureContext` — a `SavedContext` subclass holding whatever
backend-specific state (a GL texture handle, etc.) that GSG needs. All such
contexts for one GSG are owned and tracked by that GSG's single
`PreparedGraphicsObjects` instance, which also mediates release (explicit
`release_texture()` etc., or automatic eviction under the LRU above) and
batches "objects queued for release" so backend resource destruction can
happen on the draw thread. See
[PreparedGraphicsObjects.md](PreparedGraphicsObjects.md),
[SavedContext.md](SavedContext.md).

## Config variables (from `config_gobj.h`/`.cxx`)

| Variable | Purpose |
|---|---|
| `max-texture-dimension` | Cap texture width/height (downscales on load if exceeded, -1 = no cap) |
| `texture-scale` / `texture-scale-limit` / `exclude-texture-scale` | Global texture downscale factor, minimum size floor, and per-pattern exclusion list |
| `keep-texture-ram` | Keep a system-RAM copy of texture image data after GPU upload |
| `driver-compress-textures` / `driver-generate-mipmaps` | Let the driver compress textures / generate mipmaps instead of doing it in Panda |
| `vertex-buffers` / `vertex-arrays` / `display-lists` | Enable/disable each of these GSG rendering paths |
| `hardware-animated-vertices` / `display-list-animation` | Allow hardware-skinned vertex animation / animation via display lists |
| `hardware-point-sprites` / `hardware-points` / `singular-points` | Point-rendering capability toggles |
| `matrix-palette` | Enable hardware matrix-palette skinning |
| `connect-triangle-strips` / `preserve-triangle-strips` | Triangle strip optimization behavior |
| `dump-generated-shaders` / `cache-generated-shaders` | Debug-dump auto-generated shader source / cache generated shaders to disk |
| `vertices-float64` | Default `NT_stdfloat` to double- instead of single-precision |
| `vertex-column-alignment` | Byte alignment enforced for vertex columns |
| `vertex-animation-align-16` | 16-byte-align vertex animation data (SSE requirement) |
| `textures-power-2` / `textures-square` | Auto-pad textures to power-of-2 / square dimensions (enum: none/down/up) |
| `textures-auto-power-2` | Legacy bool alias for the above |
| `textures-header-only` | Load only texture headers (dimensions/format), not pixel data — for asset auditing |
| `simple-image-size` / `simple-image-threshold` | Size of the fallback low-res "simple RAM image" and the size threshold that triggers generating one |
| `geom-cache-size` / `geom-cache-min-frames` | `GeomCacheManager` cache capacity and minimum-age-before-eviction |
| `released-vbuffer-cache-size` / `released-ibuffer-cache-size` | Retained-for-reuse pool size for released vertex/index buffers |
| `default-near` / `default-far` / `lens-far-limit` / `default-fov` | Default `Lens` clip planes and field of view |
| `default-iod` / `default-converge` / `default-keystone` | Default stereo interocular distance / convergence / keystone correction |
| `vertex-save-file-directory` / `vertex-save-file-prefix` | Where/how `VertexDataSaveFile` pages vertex data out to disk |
| `vertex-data-small-size` | Threshold below which a `GeomVertexArrayData` is never paged out |
| `vertex-data-page-threads` | Thread count for async vertex data paging |
| `graphics-memory-limit` | Soft cap driving the GPU-residency LRU eviction target |
| `sampler-object-limit` | Max cached `SamplerContext` objects |
| `adaptive-lru-weight` / `adaptive-lru-max-updates-per-frame` | `AdaptiveLru` eviction-scoring weight and per-frame update budget |
| `async-load-delay` | Artificial delay injected into async texture/model loads, for testing |
| `lens-geom-segments` | Tessellation detail when building a visualization `Geom` for a `Lens` frustum |
| `stereo-lens-old-convergence` | Compatibility toggle for pre-1.9 stereo convergence math |
| `basic-shaders-only` | Restrict to a minimal shader feature subset (compatibility mode) |
| `cg-glsl-version` / `glsl-preprocess` / `glsl-include-recursion-limit` | Cg-to-GLSL target version, whether Panda preprocesses `#include`/`#pragma` in GLSL, and its recursion cap |

## See also

- [pgraph](../pgraph/README.md) — scene graph consuming `Geom`/`Texture`/
  `Shader` via `GeomNode`/`RenderAttrib`
- [display](../display/README.md) — `GraphicsStateGuardian` is what actually
  calls `prepare_now()`/issues GPU commands against the `*Context` objects
  defined here
- [event](../event/README.md) — `AsyncTask` base for
  `AnimateVerticesRequest`/`TextureReloadRequest`
