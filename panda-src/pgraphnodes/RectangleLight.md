# RectangleLight

**Source:** `panda/src/pgraphnodes/rectangleLight.h` (+ `.I`, `.cxx`)
**Inherits:** [LightLensNode](LightLensNode.md)
**Inherited by:** (none)

*Since 1.10.0.* An area light shaped as an axis-aligned rectangle, emitting
from its surface along the positive Y axis. Unlike `PointLight`/`Spotlight`,
it has no attenuation/specular-color/exponent controls of its own — its
surface area (driven by the node's scale) and orientation are what shape
its contribution, computed by the shader that consumes it (typically the
auto shader generator, see [ShaderGenerator](ShaderGenerator.md)).

## Behavior notes

- **`get_specular_color()` always returns `get_color()`** — unlike every
  other light in this module, `RectangleLight` has no independent specular
  color override at all (no `set_specular_color()`/`_has_specular_color`
  flag exists on this class).
- **`bind()` is an empty no-op** (`void RectangleLight::bind(...) {}` —
  literally an empty function body). Every other light's `bind()` forwards
  to `gsg->bind_light(this, light, light_id)` to register itself with the
  GSG's fixed-function-style per-light-id binding slot; `RectangleLight`
  skips this entirely, meaning it can't participate in that binding path at
  all — it's only usable through render paths that don't rely on
  `bind()` (in practice, the shader-based auto-generated lighting path).
  Worth knowing if you see a `RectangleLight` silently fail to light
  anything under an older/fixed-function-style renderer.
- Only new state beyond the `LightLensNode` base is `_max_distance` (hard
  falloff cutoff, default +infinity) — no constant/linear/quadratic
  attenuation terms exist on this class, since a rectangular area light's
  falloff is computed geometrically by the consuming shader rather than via
  the point-light attenuation equation.

## API

| Method | Notes |
|---|---|
| `RectangleLight(const std::string &name)` | Constructor. |
| `const LColor &get_specular_color() const` (final) | Always equals `get_color()` — no override mechanism. |
| `PN_stdfloat get_max_distance() const` / `set_max_distance(...)` | Hard falloff cutoff; default +infinity. |
| `int get_class_priority() const` | Returns `CP_area_priority`. |

## Usage

```cpp
PT(RectangleLight) rlight = new RectangleLight("area");
rlight->set_color(LColor(1, 1, 1, 1));
NodePath rlnp = render.attach_new_node(rlight);
rlnp.set_scale(2, 1, 1);  // shapes the rectangle's extent
render.set_light(rlnp);
```

## See also

- [LightLensNode](LightLensNode.md) — immediate base
- [ShaderGenerator](ShaderGenerator.md) — the render path that actually consumes this light, given `bind()` is a no-op
