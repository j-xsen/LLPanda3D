# DirectionalLight

**Source:** `panda/src/pgraphnodes/directionalLight.h` (+ `.I`, `.cxx`)
**Inherits:** [LightLensNode](LightLensNode.md) > `Light`, `Camera`

A light shining from infinitely far away in a single direction, with no
position and no distance falloff — like sunlight. Its `Lens` is an
`OrthographicLens` (parallel projection, matching the physical model of
parallel light rays), which also makes it the natural choice for a
shadow-casting sun light, since an orthographic shadow frustum avoids the
perspective-projection shadow-mapping artifacts a `PerspectiveLens` would
introduce.

## Behavior notes

- **`get_vector_to_light()` ignores the query point entirely** — it always
  returns `-direction` transformed into object space, regardless of
  `from_object_point`, since every point in the scene sees the same
  incoming light direction for a directional light. This is the defining
  behavioral difference from [PointLight](PointLight.md)'s version, which
  computes an actual point-to-point vector.
- **`_point`/`_direction` are both stored and transformed by `xform()`**,
  even though only `_direction` is physically meaningful for lighting —
  `_point` exists purely as a visualization anchor (where to draw the
  light's debug gizmo), not something shaders sample.
- **Constructor sets `interocular_distance` to 0 on its lens** — disabling
  stereo offset on the light's own `Lens`, since a light's projection
  should be monoscopic even if the scene camera is stereo.
- **`_has_specular_color` bam-version-gates.** Files saved before minor
  version 39 don't store this flag and are read back assuming
  `_has_specular_color = true` (older behavior always had an explicit
  specular color) — a compatibility shim worth knowing if you're debugging
  lighting differences between old and new `.bam` assets.

## API

| Method | Notes |
|---|---|
| `DirectionalLight(name)` | Constructor; uses an `OrthographicLens` |
| `get_specular_color()` / `set_specular_color(color)` / `clear_specular_color()` | `final` override; falls back to diffuse color if never set |
| `get_point()` / `set_point(point)` | Visualization-only anchor point, not used in lighting math |
| `get_direction()` / `set_direction(direction)` | The light ray direction (object-space) |
| `get_class_priority()` | Returns `CP_directional_priority` |

Shadow/frustum API inherited from [LightLensNode](LightLensNode.md).

## Usage

```cpp
PT(DirectionalLight) dlight = new DirectionalLight("sun");
dlight->set_color(LColor(1, 1, 0.9f, 1));
dlight->set_direction(LVector3(-1, -1, -2));
NodePath dlnp = render.attach_new_node(dlight);
render.set_light(dlnp);
```

## See also

- [LightLensNode](LightLensNode.md) — base class, shadow/frustum API
- [PointLight](PointLight.md) — position-based sibling with real distance falloff
- [../gobj/OrthographicLens.md](../gobj/OrthographicLens.md)
