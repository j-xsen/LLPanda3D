# RectangleLight

**Source:** `panda/src/pgraphnodes/rectangleLight.h` (+ `.I`, `.cxx`)
**Inherits:** [LightLensNode](LightLensNode.md) > `Light`, `Camera`

*(since 1.10.0)* An area light shaped as an axis-aligned rectangle,
emitting along its local +Y axis. Unlike the other lights in this module,
`RectangleLight` has essentially no configurable shape parameters of its
own beyond `max_distance` — its rectangle's size/orientation comes from
however the node itself is scaled/transformed in the scene graph, not from
a dedicated width/height member.

## Behavior notes

- **`bind()` is an empty no-op** (`{}` — doesn't even call
  `gsg->bind_light()` the way every other concrete light's `bind()` does).
  Area lights aren't part of the legacy fixed-function/per-vertex light
  array binding path at all — a `RectangleLight` only does anything when
  consumed by a physically-based [ShaderGenerator](ShaderGenerator.md)
  pipeline that explicitly knows how to sample rectangular area-light
  contributions; attaching one under a renderer that doesn't have that
  auto-shader support silently does nothing.
- **No `_attenuation`/`_exponent`/`_point` members at all** — of the
  `CData`-cycled state on other lights, `RectangleLight` keeps only
  `_max_distance`; its actual falloff/area-integration math is presumably
  computed by the shader from the light's world-space transform/scale
  directly, not from stored per-light coefficients.
- **`get_specular_color()` is declared `final` but there's no
  corresponding `set_specular_color()`/`clear_specular_color()` in the
  header** — unlike every other `LightLensNode` subclass, which exposes
  all three. Whatever specular color this returns comes from the base
  `Light` class's default/diffuse-color fallback rather than anything
  settable specifically on `RectangleLight`; likely just an
  under-filled-out API rather than an intentional restriction.

## API

| Method | Notes |
|---|---|
| `RectangleLight(name)` | Constructor |
| `get_specular_color()` | `final`; no matching setter on this class |
| `get_max_distance()` / `set_max_distance(d)` | Hard falloff cutoff |
| `get_class_priority()` | Returns `CP_area_priority` |

## Usage

```cpp
NodePath render = window->get_render();  // WindowFramework* window
PT(RectangleLight) rlight = new RectangleLight("panel");
rlight->set_color(LColor(1, 1, 1, 1));
NodePath rlnp = render.attach_new_node(rlight);
rlnp.set_scale(2, 1, 1);  // 2x1 unit rectangle, emits along +Y
render.set_light(rlnp);
```

## See also

- [LightLensNode](LightLensNode.md) — base class
- [ShaderGenerator](ShaderGenerator.md) — the only consumer that actually shades this light
