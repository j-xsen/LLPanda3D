# LightLensNode

**Source:** `panda/src/pgraphnodes/lightLensNode.h` (+ `.I`, `.cxx`)
**Inherits:** `Light` (external, [../pgraph/Light.md](../pgraph/Light.md)), `Camera` (external, [../pgraph/Camera.md](../pgraph/Camera.md)) **Inherited by:** [DirectionalLight](DirectionalLight.md), [PointLight](PointLight.md), [Spotlight](Spotlight.md), [RectangleLight](RectangleLight.md)

Base for every light that needs a direction and/or shaped falloff volume —
which is every concrete light except [AmbientLight](AmbientLight.md). It
inherits `Camera` (not `LensNode` directly, despite the class name's
implication — the header comment calls this out explicitly) purely to reuse
`Camera`/`Lens`'s frustum machinery: a `Spotlight`'s cone shape *is* its
`Lens`'s frustum, and shadow mapping renders the scene from the light's
point of view through that same `Lens`, exactly like an ordinary scene
camera would. See the
[pgraphnodes README](README.md#lights-lightnode-vs-lightlensnode) for the
high-level rationale.

## Behavior notes

- **The `Camera` inherited from here "serves no purpose unless shadows are
  enabled"** (per the header comment) — for a non-shadow-casting light,
  the `Lens`/frustum machinery still defines the light's directional shape
  (used by [Spotlight](Spotlight.md)'s cone, [PointLight](PointLight.md)'s
  six cube-face lenses, etc.) even though no actual `GraphicsOutput`
  render happens; it only becomes an active camera-like render target once
  `set_shadow_caster(true)` is called (`set_active(caster)`).
- **`set_shadow_caster()` lazily creates a shadow buffer/texture** sized by
  `_sb_size` (default 512×512) and sorted by `_sb_sort` (default `-10`,
  i.e. rendered early). Changing the buffer size or turning shadows off
  calls `clear_shadow_buffers()`, which explicitly removes the
  `GraphicsOutput` windows and clears the shadow map texture to all-white
  first — "so that any shaders that might still be using it will see the
  shadows being disabled" rather than sampling stale/garbage depth data.
- **Enabling shadows requires the shader generator to be active on the
  scene** — a `LightLensNode`'s shadow buffer only gets populated and
  sampled by shaders that [ShaderGenerator](ShaderGenerator.md)
  auto-generates; there's no legacy fixed-function shadow path here.
- **`_used_by_auto_shader` gates automatic shader regeneration.**
  `mark_used_by_auto_shader()` (called by `ShaderGenerator` when it
  actually consumes this light) sets a flag; if `set_shadow_caster()` is
  later called with a *different* caster value while that flag is set,
  it calls `GraphicsStateGuardianBase::mark_rehash_generated_shaders()` so
  any auto-generated shader referencing this light gets rebuilt to
  add/remove the shadow-sampling code. This mirrors the analogous flag on
  `Material` (see [../gobj/Material.md](../gobj/Material.md)) — but unlike
  `Material::is_used_by_auto_shader()`, which reads the *wrong* flag bit
  (a documented bug there), this class's own
  `_used_by_auto_shader`/`mark_used_by_auto_shader()` pair is internally
  consistent (single bool, no bitmask mismatch).
- **`attrib_ref()`/`attrib_unref()` track how many live `LightAttrib`s
  reference this node**, via an atomic counter — when the count drops to
  zero (the light is no longer attached to *any* `RenderState`), shadow
  buffers are torn down automatically, since a `GraphicsOutput`'s camera
  reference would otherwise keep this node (and transitively its shadow
  buffer's owning `DisplayRegion`) alive in a reference cycle. The
  destructor asserts this counter is exactly zero, flagging a
  `LightAttrib` ref-counting bug if it ever isn't.
- **`PointLight`'s shadow map is a cube map**, not a 2-D texture —
  `setup_shadow_map()` is virtual specifically so `PointLight` can override
  it (see [PointLight](PointLight.md)) to allocate omnidirectional shadow
  storage instead of this base's single 2-D depth texture.
- **Copy construction does not copy `_used_by_auto_shader` or
  `_attrib_count`** — both reset to `false`/`0` on a copy, since those
  track this *specific* node instance's live GPU/shader state, not
  something meaningfully duplicable.

## API

| Method | Notes |
|---|---|
| `LightLensNode(name, lens = new PerspectiveLens())` | Constructor; default initial state culls backfaces and disables color writes (used when rendering into a shadow buffer) |
| `set_shadow_caster(bool)` | Enable/disable shadows, keeping current buffer size |
| `set_shadow_caster(bool, xsize, ysize, sort = -10)` | Enable/disable with explicit shadow buffer size/sort |
| `is_shadow_caster()` | Current state |
| `get_shadow_buffer_sort()` | Current render-sort of the shadow buffer |
| `get_shadow_buffer_size()` / `set_shadow_buffer_size(size)` | Shadow map texture dimensions |
| `get_shadow_buffer(gsg)` | Debug-only lookup of an already-created per-GSG shadow buffer; returns `nullptr` if none exists yet (never creates one) |
| `has_specular_color()` | Whether an explicit specular color was set (vs. defaulting to the diffuse color) |
| `mark_used_by_auto_shader()` *(internal)* | Called by `ShaderGenerator` when it references this light |

Color/attenuation/direction API varies per concrete subclass — see
[DirectionalLight](DirectionalLight.md), [PointLight](PointLight.md),
[Spotlight](Spotlight.md), [RectangleLight](RectangleLight.md).

## See also

- [../pgraph/Camera.md](../pgraph/Camera.md), [../pgraph/Light.md](../pgraph/Light.md) — external bases
- [ShaderGenerator](ShaderGenerator.md) — consumes shadow buffers and `_used_by_auto_shader`
- [../gobj/Material.md](../gobj/Material.md) — analogous but buggy auto-shader-usage flag on `Material`
