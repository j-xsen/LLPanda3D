# SphereLight

**Source:** `panda/src/pgraphnodes/sphereLight.h` (+ `.I`, `.cxx`)
**Inherits:** [LightLensNode](LightLensNode.md) > [PointLight](PointLight.md) > `Light`, `Camera`

*(since 1.10.0)* A [PointLight](PointLight.md) with a physical radius
instead of being an infinitely thin point — the light-emitting surface is
modeled as a sphere. This is a one-level-deeper subclass (`PointLight`,
not `LightLensNode`, is its direct base), giving it every `PointLight`
feature (six-face cube-map shadow setup, quadratic attenuation, `_point`
position) plus a `_radius`.

## Behavior notes

- **`_radius` affects shading (soft shadow penumbra size / specular
  highlight size in a physically-based shader) rather than attenuation
  math** — the quadratic attenuation formula itself is unchanged from
  `PointLight`; `_radius` is a separate parameter a PBR-style
  [ShaderGenerator](ShaderGenerator.md) implementation can use to compute
  approximate soft-shadow penumbras or non-point specular highlights,
  which a true point light (radius 0) can't produce.
- **`xform()`'s radius scaling only accounts for uniform-ish scale along
  one axis** — it transforms a unit-length-`_radius` vector along local Z
  by the matrix and takes the resulting vector's length as the new radius
  (`mat.xform_vec(LVector3(0,0,radius)).length()`). Under a non-uniform
  scale, this gives a radius based specifically on the Z-axis scale factor,
  not an average or a proper ellipsoid-preserving transform — same
  approximate-scaling tradeoff `LODNode::xform()` makes with switch
  distances.

## API

| Method | Notes |
|---|---|
| `SphereLight(name)` | Constructor |
| `get_radius()` / `set_radius(radius)` | The sphere's physical radius |

All other API (attenuation, max distance, position, shadow casting) is
inherited unchanged from [PointLight](PointLight.md).

## Usage

```cpp
NodePath render = window->get_render();  // WindowFramework* window
PT(SphereLight) slight = new SphereLight("bulb");
slight->set_color(LColor(1, 1, 1, 1));
slight->set_radius(0.5f);
NodePath slnp = render.attach_new_node(slight);
render.set_light(slnp);
```

## See also

- [PointLight](PointLight.md) — direct base class
- [LightLensNode](LightLensNode.md) — shared light-with-lens base
