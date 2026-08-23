# Spotlight

**Source:** `panda/src/pgraphnodes/spotlight.h` (+ `.I`, `.cxx`)
**Inherits:** [LightLensNode](LightLensNode.md)
**Inherited by:** (none)

A light originating from a single point, shining in a particular direction
with a cone-shaped falloff. The cone shape is defined entirely by the
`Lens` this node carries (inherited from `LightLensNode`/`Camera`) — since
any `Lens` subclass can be installed, a spotlight's cone can have any FOV,
aspect ratio, or even be a non-perspective shape if you install an unusual
lens. (The class is named `Spotlight`, one word, deliberately — not
`SpotLight`.)

## Behavior notes

- **`get_exponent()`/`set_exponent()`** control falloff *within* the cone:
  the light is attenuated by `cos(angle)^exponent`, where `angle` is
  between the light's aim direction and the direction to the lit point —
  independent of the lens's actual field-of-view, so a wide-FOV lens with a
  high exponent still produces a tightly focused-looking hotspot. Default
  is `50.0`.
- **`get_vector_to_light()` always returns `false`** — unlike
  `DirectionalLight`/`PointLight`, `Spotlight` doesn't implement this
  (legacy fixed-function-style) vector computation at all; the comment
  block above it is boilerplate copy-pasted from the other lights and
  doesn't actually describe `Spotlight`'s behavior.
- **`make_spot()` is a static utility**, unrelated to any particular
  `Spotlight` instance — it renders a `PNMImage` with a circular falloff
  (via `PNMImage::render_spot()`) and wraps it as a `Texture`, useful for
  faking a spotlight's visual footprint by projecting this texture onto
  geometry (e.g. via `NodePath::project_texture()`) instead of enabling
  real per-pixel lighting — "a cheesy way," per the source comment.
- Reuses [PointLight](PointLight.md)'s three-term attenuation model
  (`get_attenuation()`) and `max_distance` hard cutoff — same fields,
  independently declared here rather than inherited (since `Spotlight`
  doesn't derive from `PointLight`).
- `fill_viz_geom()`/`get_viz_state()` build the little frustum-wireframe
  visualization geometry shown when the scene graph is rendered in debug
  visualization mode — delegates to `Lens::make_geometry()`.
- `get_class_priority()` returns `CP_spot_priority`.

## API

| Method | Notes |
|---|---|
| `Spotlight(const std::string &name)` | Constructor. |
| `PN_stdfloat get_exponent() const` (final) / `set_exponent(...)` | Cone falloff sharpness; default `50.0`. |
| `const LColor &get_specular_color() const` (final) / `set_specular_color(...)` / `clear_specular_color()` | Custom specular color, defaults to `get_color()`. |
| `const LVecBase3 &get_attenuation() const` (final) / `set_attenuation(...)` | Constant/linear/quadratic falloff, same model as `PointLight`. |
| `PN_stdfloat get_max_distance() const` / `set_max_distance(...)` | Hard falloff cutoff; default +infinity. |
| `static PT(Texture) make_spot(int pixel_width, PN_stdfloat full_radius, LColor &fg, LColor &bg)` | Renders a standalone circular-spot texture, independent of any Spotlight instance. |
| `int get_class_priority() const` | Returns `CP_spot_priority`. |

## Usage

```cpp
PT(Spotlight) spot = new Spotlight("spot");
spot->set_color(LColor(1, 1, 1, 1));
spot->get_lens()->set_fov(30);
spot->set_exponent(20);
NodePath spotnp = render.attach_new_node(spot);
render.set_light(spotnp);
```

## See also

- [LightLensNode](LightLensNode.md) — immediate base
- [PointLight](PointLight.md) — same attenuation model, no cone
- [Lens](../gobj/Lens.md) — defines the cone/frustum shape
