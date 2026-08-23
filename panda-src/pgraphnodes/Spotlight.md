# Spotlight

**Source:** `panda/src/pgraphnodes/spotlight.h` (+ `.I`, `.cxx`)
**Inherits:** [LightLensNode](LightLensNode.md) > `Light`, `Camera`

A light originating from a single point, shining in a particular direction
with a cone-shaped falloff. The cone shape is defined entirely by the
inherited `Lens` (see [LightLensNode](LightLensNode.md)) — since a
`Spotlight` uses a single ordinary `Lens` (typically a `PerspectiveLens`),
it can have any property a camera lens can (arbitrary FOV, off-axis
projection, etc.), not just a symmetric circular cone. Named `Spotlight`,
not `SpotLight`, because — per the source comment — "spotlight" is one
English word.

## Behavior notes

- **The falloff exponent shapes edge softness, separate from the cone
  angle.** `_exponent` controls how sharply intensity drops off toward the
  cone edge (higher = tighter, more concentrated hotspot); the cone *angle*
  itself comes from the `Lens`'s field of view, not from any `Spotlight`
  member — two independent knobs.
- **Attenuation and max-distance work identically to `PointLight`**
  (quadratic `(constant, linear, quadratic)` coefficients, separate hard
  `max_distance` cutoff) — `Spotlight` just adds the cone-specific
  `exponent` on top.
- **`get_vector_to_light()` always returns `false`.** Unlike
  `DirectionalLight`/`PointLight`, a `Spotlight`'s implementation is a
  stub that never computes a result — callers must check the return value;
  this looks like an intentional not-yet-implemented gap (the cone
  direction plus position would need combining, similar to `PointLight`,
  but nobody wrote it) rather than a documented design choice.
- **`make_spot()` is a static utility unrelated to actual lighting** — it
  procedurally renders a circular gradient `PNMImage` (radial falloff from
  `fg` to `bg` color) and wraps it as a `Texture`, for *projecting* a
  cheap "fake spotlight" texture onto geometry (e.g. via
  `NodePath::project_texture()`) instead of enabling a real light. Channel
  count is chosen automatically: grayscale if `fg`/`bg` are each
  colorless, +1 channel if either has non-1.0 alpha.
- **Shadow visualization (`fill_viz_geom()`) reuses the `Lens`'s own
  `make_geometry()`** to draw the actual frustum shape as debug geometry,
  rather than Spotlight computing its own cone mesh — so the visualization
  automatically matches whatever `Lens` subclass/parameters are actually
  set, including non-standard lenses.
- Constructor sets `interocular_distance` to 0 on its single lens, same as
  `DirectionalLight` — the light's own projection is never stereo even if
  the scene camera is.

## API

| Method | Notes |
|---|---|
| `Spotlight(name)` | Constructor |
| `get_exponent()` / `set_exponent(exponent)` | `final`; falloff sharpness toward the cone edge |
| `get_specular_color()` / `set_specular_color(color)` / `clear_specular_color()` | `final` override |
| `get_attenuation()` / `set_attenuation(LVecBase3)` | `final`; `(constant, linear, quadratic)` |
| `get_max_distance()` / `set_max_distance(d)` | Hard falloff cutoff |
| `get_class_priority()` | Returns `CP_spot_priority` |
| `Spotlight::make_spot(pixel_width, full_radius, fg, bg)` *(static)* | Generates a projectable "fake spotlight" gradient `Texture`, unrelated to real lighting |

Cone shape/FOV comes entirely from the inherited `Lens` (see
[LightLensNode](LightLensNode.md), `Lens.get_lens()`/`set_lens()`).

## Usage

```cpp
NodePath render = window->get_render();  // WindowFramework* window
PT(Spotlight) slight = new Spotlight("flashlight");
slight->set_color(LColor(1, 1, 1, 1));
slight->get_lens()->set_fov(30);
slight->set_exponent(60.0f);
NodePath slnp = render.attach_new_node(slight);
render.set_light(slnp);
```

## See also

- [LightLensNode](LightLensNode.md) — base class, `Lens`/shadow API
- [PointLight](PointLight.md) — omnidirectional sibling
- [../gobj/Lens.md](../gobj/Lens.md), [../gobj/PerspectiveLens.md](../gobj/PerspectiveLens.md)
