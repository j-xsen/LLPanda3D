# PointLight

**Source:** `panda/src/pgraphnodes/pointLight.h` (+ `.I`, `.cxx`)
**Inherits:** [LightLensNode](LightLensNode.md) > `Light`, `Camera` **Inherited by:** [SphereLight](SphereLight.md)

A light originating from a single point in space and shining equally in
all directions, with distance-based attenuation. Unlike
[Spotlight](Spotlight.md) (one cone-shaped `Lens`), a `PointLight` needs to
cover the full sphere around its position — it does this with **six**
90°×90° `PerspectiveLens`es, one per cube face, giving it (via the inherited
multi-lens `Camera`/`LensNode` machinery) an omnidirectional view for
cube-map shadow rendering.

## Behavior notes

- **Attenuation is a 3-component quadratic polynomial**
  (`_attenuation = (constant, linear, quadratic)`), the classic
  `1 / (Kc + Kl*d + Kq*d²)` falloff formula — set via `set_attenuation()`
  as an `LVecBase3`, consumed by [ShaderGenerator](ShaderGenerator.md)
  (and any legacy fixed-function lighting path) when computing per-pixel
  intensity at a given distance `d` from `_point`.
- **`get_max_distance()` is a hard cutoff, separate from attenuation** —
  even with nonzero attenuation making the light theoretically extend to
  infinity, `max_distance` (default `+inf`, only round-tripped in `.bam`
  files at minor version ≥ 41) clips the light's influence entirely past a
  fixed radius for performance, independent of how dim the attenuation
  curve makes it at that distance.
- **`get_vector_to_light()` returns a real point-to-point vector** (unlike
  [DirectionalLight](DirectionalLight.md)'s constant direction) —
  `_point` transformed into object space, minus the query vertex position.
- **Shadow map is a cube map, sized/square-enforced.** `setup_shadow_map()`
  overrides the base `LightLensNode` version to call
  `Texture::setup_cube_map()` instead of `setup_2d_texture()`; if
  `_sb_size`'s width and height differ (settable via the base class's
  `set_shadow_caster(caster, xsize, ysize)`), it logs an error since a cube
  map shadow buffer must be square — a `PointLight`-specific validation
  the base class doesn't need. A source comment also notes cube map shadow
  filtering doesn't work correctly under Cg, so filtering falls back to
  `FT_linear` instead of the shadow-comparison filter mode
  [LightLensNode](LightLensNode.md) uses for its single 2-D shadow map.
- **`xform()` only transforms `_point`**, not the six per-face lenses —
  their view directions are fixed relative to the node's local axes by
  construction and rotate implicitly with the node's own transform in the
  scene graph, so there's nothing extra to update on an `xform()` call
  beyond the position anchor.

## API

| Method | Notes |
|---|---|
| `PointLight(name)` | Constructor; sets up 6 cube-face `PerspectiveLens`es (90°×90° each) |
| `get_specular_color()` / `set_specular_color(color)` / `clear_specular_color()` | `final` override |
| `get_attenuation()` / `set_attenuation(LVecBase3)` | `final` override; `(constant, linear, quadratic)` coefficients |
| `get_max_distance()` / `set_max_distance(d)` | Hard falloff cutoff distance, default `+inf` |
| `get_point()` / `set_point(point)` | Light position (object space) |
| `get_class_priority()` | Returns `CP_point_priority` |

Shadow/frustum API inherited from [LightLensNode](LightLensNode.md)
(`setup_shadow_map()` overridden for cube-map storage).

## Usage

```cpp
NodePath render = window->get_render();  // WindowFramework* window
PT(PointLight) plight = new PointLight("bulb");
plight->set_color(LColor(1, 1, 1, 1));
plight->set_attenuation(LVecBase3(1, 0, 0.02f));
NodePath plnp = render.attach_new_node(plight);
plnp.set_pos(0, 0, 10);
render.set_light(plnp);
```

## See also

- [LightLensNode](LightLensNode.md) — base class, shadow/frustum API
- [SphereLight](SphereLight.md) — subclass adding a physical radius
- [Spotlight](Spotlight.md) — single-cone sibling
