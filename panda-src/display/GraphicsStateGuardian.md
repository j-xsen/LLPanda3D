# GraphicsStateGuardian

**Source:** `panda/src/display/graphicsStateGuardian.h` (+ `.I`, `.cxx`)
**Inherits:** GraphicsStateGuardianBase (external, `panda/src/pgraphnodes`/gobj area — not part of this module) **Inherited by:** every concrete 3-D API backend (`GLGraphicsStateGuardian`, DirectX GSGs, TinyDisplay, etc. — all outside `panda/src/display`)

The abstract base class ("GSG") that owns all communication with one instance of a rendering backend context — one GSG per active graphics context in the system, normally one per open window/buffer sharing a context. Its job is threefold: (1) report hardware/API capabilities (max texture stages, shader model, supported texture/buffer formats...) so the rest of Panda can adapt, (2) prepare/release GPU-side resources for `Texture`, `Geom`, `Shader`, vertex/index/shader buffers via `*Context` objects, and (3) drive the actual per-frame render sequence (`begin_frame`/`begin_scene`/draw calls/`end_scene`/`end_frame`) and apply `RenderState`/`TransformState` to the hardware. Base-class method bodies here are almost all either trivial accessors backed by protected member fields, or no-op/return-false stubs that a real backend overrides — this class defines the *contract*, not the rendering itself.

## Behavior notes

- **Capability fields are all initialized to "nothing supported" in the
  constructor**, then overwritten by a subclass's `reset()` once it queries
  the real hardware/API. Until `reset()` has run (`needs_reset()`/`_needs_reset`
  starts `true`), fields like `get_max_texture_stages()` are not meaningful —
  every getter's doc comment repeats "may not be meaningful until the graphics
  context has been fully created." `reset_if_new()` is the idiom subclasses
  use to lazily call `reset()` exactly once.
- **`get_max_texture_stages()` and `get_max_color_targets()` clamp against
  config variables**, not just the hardware limit: if `max-texture-stages`
  (or `max-color-targets`) is configured `> 0`, the getter returns
  `min(hardware_limit, config_value)` — the config variable can only lower
  the effective limit, never raise it beyond what the GSG actually supports.
- **`get_copy_texture_inverted()` lets a config variable override the GSG's
  self-reported value**: if `copy-texture-inverted` was explicitly set in a
  prc file, that wins; otherwise the GSG's own `_copy_texture_inverted` flag
  (set by the subclass in `reset()`) is used.
- **`get_geom_munger()` caches per-`RenderState` mungers keyed by a
  process-wide unique GSG `_id`** (assigned in the constructor from a static
  counter, never a raw pointer — pointers get reused after a GSG is
  destroyed, ids never do). It first checks `RenderState::_last_mi` as a
  fast-path guess (states tend to be revisited by the same GSG many times a
  frame) before falling back to a full map lookup; a munger found but no
  longer `is_registered()` is evicted and a fresh one built via the virtual
  `make_geom_munger()` (which the base class stubs to return `nullptr` —
  every real GSG must override it, typically returning a `StandardMunger` or
  subclass).
- **The `begin_frame`/`begin_scene`/`draw_*`/`end_scene`/`end_frame` sequence
  has a strict nesting contract**, enforced only by convention/asserts, not
  by the type system: `begin_frame()` returns false (skip the whole frame,
  no `end_frame()` call) if `_needs_reset` is still set; within a frame,
  `begin_scene()`/`end_scene()` brackets exactly one `DisplayRegion`'s
  render pass, and `begin_draw_primitives()`/`end_draw_primitives()` brackets
  a batch of `draw_triangles()`/`draw_tristrips()`/etc. calls for one `Geom`.
  `end_scene()` explicitly re-disables every light and clip plane that was
  enabled that scene (forcing rebind next scene, since positions/params may
  have changed) and resets `_state_rs` to empty so `set_state_and_transform()`
  reloads everything for the next scene.
- **`begin_frame()` unconditionally resets `_state_rs` to empty at the start
  of every frame**, even though this costs extra state-reapplication
  overhead — the comment explains this guards against attribute-pointer-
  stable-but-value-changed cases (a `Texture` whose image changed in place,
  a `Light`'s color changed) that a plain pointer-equality state cache would
  miss.
- **`do_issue_light()`/`do_issue_clip_plane()` implement generic ID-slot
  assignment logic in the base class** (not per-backend): both walk the
  active `LightAttrib`/`ClipPlaneAttrib`, filter to the hardware's max count
  (`_max_lights`/`_max_clip_planes`), and call the backend's
  `enable_lighting()`/`bind_light()`/`enable_light()` (or the clip-plane
  equivalents) only for the delta from last scene — reusing IDs where
  possible, only calling `begin_bind_lights()`/`end_bind_lights()` once
  per scene around the whole batch. `do_issue_light()` also folds ambient
  lights into a single `set_ambient_light()` color contribution rather than
  binding them as hardware lights.
- **`do_issue_color()`/`do_issue_color_scale()`/`determine_light_color_scale()`
  implement the "fake color via lighting/texture" trick** described in the
  `color-scale-via-lighting`/`alpha-scale-via-texture` config variables (see
  [README.md](README.md)): rather than munging every vertex's color, these
  methods compute a synthetic material-force-color and/or light-color-scale
  that produces the same visual effect through the existing lighting
  pipeline, clearing the relevant `_state_mask` bits so the backend re-sends
  material/light state. `_has_texture_alpha_scale` similarly tracks whether
  an extra `TextureStage` (`get_alpha_scale_texture_stage()`, a process-wide
  singleton `TextureStage` with sort `1000000000` so it's always last) was
  injected to fake an alpha scale.
- **`fetch_specified_value()`/`fetch_specified_part()`/`fetch_specified_member()`/
  `fetch_specified_texture()`/`fetch_ptr_parameter()` are the shader-input
  plumbing** invoked by generated shader code (`Shader::ShaderMatSpec` /
  `ShaderTexSpec` / `ShaderPtrSpec`) to resolve `p3d_ModelViewMatrix`-style
  built-in shader inputs to actual matrix/texture/pointer values each frame,
  with small per-spec caches (`spec._cache[]`) invalidated via a bitmask
  (`altered`) of what changed since last evaluation. This is a large,
  mechanical `switch` over `Shader::ShaderMatInput`/`ShaderTexInput` enum
  values (matrix composition, light parameter packing, coordinate-space
  conversions) — treated here as an internal implementation detail rather
  than documented input-by-input; determining what a specific `SMO_*`/`STO_*`
  value resolves to requires reading the `switch` directly.
- **Decal rendering is a three-pass protocol** (`depth_offset_decals()` lets
  a GSG opt out in favor of a single-pass `DepthOffsetAttrib` approach
  instead): `begin_decal_base_first()` returns a `RenderState` that disables
  depth-buffer writes for drawing the base geometry, `begin_decal_nested()`
  keeps depth writes off while nested decal geometry draws on top, and
  `begin_decal_base_second()` returns a state that disables color writes
  (but explicitly leaves texturing on, for alpha-tested dual-transparency
  decals) so the base geometry can be redrawn a second time purely to
  restore the depth buffer.
- **`async_reload_texture()` de-duplicates in-flight reload requests** by
  task name (`"reload:" + texture_name`) — if a `TextureReloadRequest` for
  the same `Texture` is already queued on the `Loader`'s task manager, this
  just bumps its priority to `max(existing, requested)` and returns the
  existing task rather than double-queueing.
- **Shadow maps are cached per-light, per-GSG.** `get_shadow_map()` returns
  a shared dummy 1×1 depth texture (`get_dummy_shadow_map()`, a process-wide
  singleton per texture type, filled with clear-value 1) for lights that
  aren't shadow casters; for real shadow casters it looks up
  `light->_sbuffers[this]` (keyed by GSG pointer, since separate GSGs need
  separate shadow buffers) and lazily creates one via the virtual
  `make_shadow_buffer()` if absent — a `PointLight` gets six mono
  `DisplayRegion`s (one per cube face, `set_target_tex_page(i)`), other
  shadow-casting lights get one. `make_shadow_buffer()`'s default
  implementation opens an offscreen, refuse-window buffer via
  `GraphicsEngine::make_output()`, sized square for point lights
  (`GraphicsPipe::BF_size_square`), with depth bits from the
  `shadow-depth-bits` config variable.
- **`ensure_generated_shader()` is a no-op unless Cg support was compiled
  in** (`#ifdef HAVE_CG`) — it lazily creates a process-wide `ShaderGenerator`
  the first time an auto-shader (`ShaderAttrib::auto_shader()`) is
  encountered and the GSG supports basic shaders, then synthesizes and
  caches the generated `ShaderAttrib` on the `RenderState` itself
  (`state->_generated_shader`), invalidated by a sequence-number comparison
  (`_generated_shader_seq`) rather than by content hashing.
- **Driver-info getters (`get_driver_vendor()`, `get_driver_renderer()`,
  `get_driver_version()`, `has_extension()`, etc.) all default to empty
  string / `-1` / `false`** in the base class — only a real backend (OpenGL
  is explicitly called out as the only one currently implementing
  `has_extension()`) fills these in meaningfully.
- **`close_gsg()` deliberately does not release individual textures/geoms
  itself** — the comment explains that because a GSG may not be the
  currently-active context in its API (OpenGL keeps one active context per
  thread), explicitly releasing resources here could delete some *other*
  GSG's live objects; it just clears `_prepared_objects` (dropping Panda's
  references) and trusts the graphics API to reclaim the underlying GPU
  resources once the context itself is destroyed.

## RenderBuffer (folded in — `renderBuffer.h`)

A tiny value type: a `GraphicsStateGuardian*` paired with an `int` bitmask
of `RenderBuffer::Type` flags (`T_front_left`/`T_back_left`/`T_depth`/
`T_stencil`/`T_aux_rgba_0..3`/etc. — which layer(s) of the framebuffer an
operation should target). `GraphicsStateGuardian::get_render_buffer(buffer_type,
prop)` is the only constructor call site in this module — it masks the
requested `buffer_type` against both the `FrameBufferProperties`' actual
buffer mask and the GSG's current `_stereo_buffer_mask` (set by
`prepare_display_region()` to exclude the eye that isn't being rendered,
for red-blue/side-by-side stereo). Passed to `clear()`,
`framebuffer_copy_to_texture()`, and `framebuffer_copy_to_ram()`.

## PStatGPUTimer (folded in — `pStatGPUTimer.h`/`.I`)

An RAII PStats timer (`PStatTimer` subclass) that, in addition to CPU-side
timing, issues a GPU timer query (`GraphicsStateGuardian::issue_timer_query()`)
on construction and destruction when `gsg->get_timer_queries_active()` is
true, so PStats can report actual GPU execution time for a code region
rather than just how long it took to queue the API commands. Only usable on
the draw thread. Compiles to a near-empty CPU-only `PStatTimer` wrapper when
`DO_PSTATS` isn't defined. Purely an internal profiling tool used inside GSG
subclass implementations — not part of the app-facing API.

## API

### Capability queries (`PUBLISHED`, all effectively read-only until `reset()` has run)

| Group | Representative signatures |
|---|---|
| Vertex/primitive limits | `get_max_vertices_per_array()`, `get_max_vertices_per_primitive()`, `prefers_triangle_strips()` |
| Texture limits | `get_max_texture_stages()`, `get_max_texture_dimension()`, `get_max_3d_texture_dimension()`, `get_max_2d_texture_array_layers()`, `get_max_cube_map_dimension()`, `get_max_buffer_texture_size()` |
| Texture feature support | `get_supports_texture_combine/saved_result/dot3()`, `get_supports_3d_texture/2d_texture_array/cube_map/buffer_texture/cube_map_array()`, `get_supports_tex_non_pow2/texture_srgb()`, `get_supports_compressed_texture()` / `get_supports_compressed_texture_format(mode)` |
| Lights/clip planes | `get_max_lights()`, `get_max_clip_planes()` |
| Vertex animation | `get_max_vertex_transforms()`, `get_max_vertex_transform_indices()` |
| Shader/pipeline support | `get_supports_basic/geometry/tessellation/compute_shaders()`, `get_supports_glsl/hlsl()`, `get_shader_model()`/`set_shader_model()`, `get_supports_cg_profile(name)` |
| Misc feature flags | `get_supports_multisample()`, `get_supports_generate_mipmap()`, `get_supports_depth_texture/depth_stencil/luminance_texture/shadow_filter()`, `get_supports_sampler_objects()`, `get_supports_stencil()`/`get_supports_two_sided_stencil()`, `get_supports_geometry_instancing()`, `get_supports_indirect_draw()`, `get_supports_occlusion_query()`/`get_supports_timer_query()`, `get_max_color_targets()`, `get_supports_dual_source_blending()` |
| Driver identity | `get_driver_vendor/renderer/version()`, `get_driver_version_major/minor()`, `get_driver_shader_version_major/minor()`, `has_extension(name)` |

Every one of these also has a `MAKE_PROPERTY` exposing it Python-side as a
plain attribute (e.g. `gsg.max_texture_stages`); irrelevant for C++ callers.

### Identity / state flags

| Signature | Notes |
|---|---|
| `INLINE GraphicsPipe *get_pipe() const` | Pipe this GSG was created on. |
| `GraphicsEngine *get_engine() const` | |
| `INLINE const GraphicsThreadingModel &get_threading_model() const` | |
| `INLINE bool is_hardware() const` | |
| `INLINE void set_active(bool)` / `INLINE bool is_active() const` | Inactive GSGs render nothing; normally only toggled off after a low-level rendering problem (see `panic_deactivate()`). |
| `INLINE bool is_valid() const` / `INLINE bool needs_reset() const` | |
| `INLINE bool reset_if_new()` / `INLINE void mark_new()` / `virtual void reset()` | |
| `void set_coordinate_system(CoordinateSystem)` / `INLINE CoordinateSystem get_coordinate_system() const` | External (scene graph) coordinate system. |
| `virtual CoordinateSystem get_internal_coordinate_system() const` | The GSG's own internal convention; vertices pre-transformed to `C_clip_point` are expected in this system. |
| `INLINE void set_incomplete_render(bool)` / `get_incomplete_render()` / `get_effective_incomplete_render()` | See doc comment in `.I` — effective value also considers the current `DisplayRegion`'s flag; only meaningful during draw. |
| `INLINE void set_loader(Loader*)` / `get_loader()` | Used for `async_reload_texture()` when incomplete-render is active. |
| `INLINE void set_shader_generator(ShaderGenerator*)` / `get_shader_generator()` | |
| `virtual bool set_gamma(PN_stdfloat)` / `get_gamma()` / `virtual void restore_gamma()` | Base implementation always returns false (unsupported); stores the value regardless. |
| `INLINE void set_texture_quality_override(Texture::QualityLevel)` / `get_texture_quality_override()` | Global override, mainly for the TinyDisplay software renderer. |
| `void set_flash_texture(Texture*)` / `clear_flash_texture()` / `get_flash_texture()` | Debug-only (`#ifndef NDEBUG`); makes a texture visibly flash, color-coded by mipmap level actually sampled. OpenGL-only implementation. |

### Resource preparation (base class stubs — overridden per backend)

| Signature | Default behavior |
|---|---|
| `virtual TextureContext *prepare_texture(Texture*, int view)` | Returns `nullptr`. |
| `virtual bool update_texture(TextureContext*, bool force)` | Returns `true`. |
| `virtual void release_texture(TextureContext*)` | No-op. |
| `virtual bool extract_texture_data(Texture*)` | Returns `false`; called only via `GraphicsEngine::extract_texture_data()`. |
| `virtual SamplerContext *prepare_sampler(const SamplerState&)` / `release_sampler()` | |
| `virtual GeomContext *prepare_geom(Geom*)` / `release_geom()` | Call via `Geom::prepare()`, not directly. |
| `virtual ShaderContext *prepare_shader(Shader*)` / `release_shader()` | |
| `virtual VertexBufferContext *prepare_vertex_buffer(GeomVertexArrayData*)` / `release_vertex_buffer()` | |
| `virtual IndexBufferContext *prepare_index_buffer(GeomPrimitive*)` / `release_index_buffer()` | |
| `virtual BufferContext *prepare_shader_buffer(ShaderBuffer*)` / `release_shader_buffer()` | |
| `INLINE void release_all()` / `release_all_textures/samplers/geoms/vertex_buffers/index_buffers/shader_buffers()` | Delegate to `_prepared_objects`. |
| `virtual PreparedGraphicsObjects *get_prepared_objects()` | |

### Munger / geometry adaptation

| Signature | Notes |
|---|---|
| `virtual PT(GeomMunger) get_geom_munger(const RenderState*, Thread*)` | Cached lookup — see behavior notes. |
| `virtual PT(GeomMunger) make_geom_munger(const RenderState*, Thread*)` | Base returns `nullptr`; every real GSG overrides (typically returns `StandardMunger` — see [StandardMunger.md](StandardMunger.md)). |

### Frame / scene lifecycle

| Signature | Notes |
|---|---|
| `virtual bool begin_frame(Thread*)` | Returns `!_needs_reset`. |
| `virtual bool begin_scene()` / `virtual void end_scene()` | Bracket one `DisplayRegion`'s render pass. |
| `virtual void end_frame(Thread*)` | Flushes PStats collectors, begins the `_graphics_memory_lru` epoch. |
| `virtual void prepare_display_region(DisplayRegionPipelineReader*)` | Sets `_current_display_region`/stereo channel/color write mask for subsequent draw calls. |
| `virtual void clear_before_callback()` | Neutralizes driver state before an app callback runs (e.g. closes open GL vertex buffers). |
| `virtual void clear_state_and_transform()` | Forces full state reload next `set_state_and_transform()`. |
| `virtual void set_state_and_transform(const RenderState*, const TransformState*)` | The core per-Geom state-application entry point; base is a no-op — all real work is backend-specific. |
| `virtual void clear(DrawableRegion *clearable)` | Base no-op; real backends read the region's clear flags/colors and issue the actual clear. |
| `RenderBuffer get_render_buffer(int buffer_type, const FrameBufferProperties&)` | See RenderBuffer section above. |
| `virtual bool set_scene(SceneSetup*)` / `virtual SceneSetup *get_scene() const final` | Must be called before traversal; computes `_projection_mat` via `calc_projection_mat()` and calls `prepare_lens()`. |
| `virtual CPT(TransformState) calc_projection_mat(const Lens*)` | Base: identity for any linear lens, `nullptr` for non-linear. |
| `virtual bool prepare_lens()` | Base returns `false`. |

### Drawing (all base stubs returning `false`/no-op)

| Signature |
|---|
| `virtual bool begin_draw_primitives(const GeomPipelineReader*, const GeomVertexDataPipelineReader*, bool force)` |
| `virtual bool draw_triangles/_adj/tristrips/_adj/trifans/patches/lines/_adj/linestrips/_adj/points(const GeomPrimitivePipelineReader*, bool force)` |
| `virtual void end_draw_primitives()` |
| `virtual bool depth_offset_decals()` / `begin_decal_base_first/nested/base_second()` / `finish_decal()` | Decal 3-pass protocol; see behavior notes. |
| `virtual void dispatch_compute(int, int, int)` | Base raises an assert ("Compute shaders not supported by GSG"). |
| `virtual void begin_occlusion_query()` / `virtual PT(OcclusionQueryContext) end_occlusion_query()` | |
| `virtual PT(TimerQueryContext) issue_timer_query(int pstats_index)` | |
| `void flush_timer_queries()` | Drains completed GPU timer queries into PStats; called by `GraphicsEngine` on the draw thread. |

### Lighting / clip planes (generic base logic + backend hooks)

| Signature | Notes |
|---|---|
| `void do_issue_clip_plane()` / `void do_issue_color()` / `void do_issue_color_scale()` / `virtual void do_issue_light()` | Generic slot-assignment logic; see behavior notes. |
| `virtual void bind_light(PointLight*/DirectionalLight*/Spotlight*, const NodePath&, int light_id)` | Backend hook, called once per newly-bound light-per-frame. |
| Protected hooks: `enable_lighting()`, `set_ambient_light()`, `enable_light()`, `begin_bind_lights()`/`end_bind_lights()`, `enable_clip_planes()`, `enable_clip_plane()`, `begin_bind_clip_planes()`/`bind_clip_plane()`/`end_bind_clip_planes()` | All base no-ops, overridden per backend. |

### Shader input resolution (internal — see behavior notes)

`fetch_specified_value()`, `fetch_specified_part()`, `fetch_specified_member()`,
`fetch_specified_texture()`, `fetch_ptr_parameter()`.

### Shadow maps

| Signature | Notes |
|---|---|
| `PT(Texture) get_shadow_map(const NodePath &light_np, GraphicsOutputBase *host=nullptr)` | Per-light, per-GSG cached; see behavior notes. |
| `PT(Texture) get_dummy_shadow_map(Texture::TextureType) const` | Shared 1×1 fallback for non-shadow-casting lights. |
| `virtual GraphicsOutput *make_shadow_buffer(LightLensNode*, Texture*, GraphicsOutput *host)` | Overridable factory for the shadow-map render target. |

### Misc

| Signature | Notes |
|---|---|
| `virtual void remove_window(GraphicsOutputBase*)` | Thin forward to `GraphicsEngine::remove_window()`. |
| `PN_stdfloat compute_distance_to(const LPoint3&) const` | Camera-plane distance in the GSG's internal coordinate system. |
| `static void create_gamma_table(PN_stdfloat gamma, unsigned short *r, *g, *b)` | Fills 256-entry gamma-correction lookup tables. |
| `void traverse_prepared_textures(TextureCallback*, void *arg)` | Visits all currently-prepared textures until the callback returns false. |
| `virtual void ensure_generated_shader(const RenderState*)` | Auto-shader synthesis; see behavior notes. |
| `void set_current_properties(const FrameBufferProperties*)` | Called by the owning output before rendering into it. |
| `AsyncFuture *async_reload_texture(TextureContext*)` (protected) | See behavior notes. |
| `virtual void close_gsg()` (protected) | Called by the owning `GraphicsWindow`/`GraphicsOutput` on close. |
| `void panic_deactivate()` (protected) | Deactivates the GSG and throws `"panic-deactivate-gsg"`. |

## See also

- [StandardMunger.md](StandardMunger.md) — the munger every GSG typically creates via `make_geom_munger()`.
- [GraphicsEngine.md](GraphicsEngine.md) — drives `begin_frame()`/`render_frame()`/`end_frame()` and owns the render pipeline.
- [GraphicsOutput.md](GraphicsOutput.md), [DisplayRegion.md](DisplayRegion.md) — supply the `DrawableRegion`/`DisplayRegionPipelineReader` objects passed into `clear()`/`prepare_display_region()`.
- [GraphicsPipe.md](GraphicsPipe.md) — the pipe a GSG is created on (`get_pipe()`).
- [FrameBufferProperties.md](FrameBufferProperties.md) — passed to `get_render_buffer()` and stored via `set_current_properties()`.
