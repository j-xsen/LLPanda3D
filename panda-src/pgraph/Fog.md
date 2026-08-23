# Fog

**Source:** `panda/src/pgraph/fog.h` (+ `.I`, `.cxx`)
**Inherits:** PandaNode

Specifies how atmospheric fog effects are applied to geometry. Being a
`PandaNode`, a `Fog` can be parented anywhere in the scene graph to define
its effect relative to a particular coordinate system, similar to
[Light](Light.md) — but it isn't attached to nodes directly; it's applied
through a [FogAttrib](FogAttrib.md) referencing this node.

## Behavior notes

- Three modes: `M_linear` (`f = (end - z) / (end - start)`),
  `M_exponential` (`f = e^(-density*z)`), `M_exponential_squared`
  (`f = e^((-density*z)^2)`). Default mode is `M_linear` with onset
  `(0,0,0)` → opaque `(0,100,0)`, color white, exp density `0.5`.
- **Exponential fog is always camera-relative** — it doesn't matter where
  the `Fog` node is parented (or whether it's parented at all). **Linear
  fog is spatial**: onset/opaque points are offsets along the local
  forward axis (Y), so parenting the `Fog` node somewhere in the graph
  localizes the fog to a region rather than always following the camera.
  An unparented `Fog` in linear mode behaves as if parented to the camera
  (traditional camera-relative fog).
- `set_linear_range(onset, opaque)` and `set_exp_density()` each
  **implicitly switch `_mode`** — the former forces `M_linear`, the latter
  forces `M_exponential` if currently `M_linear`. Order of calls matters if
  you're mixing linear and exponential setup on the same node.
- **The parallel-fog-vector problem**: real fog hardware only supports a
  1-D distance-from-camera-plane falloff, but linear in-world fog defines
  fog along an arbitrary 3-D vector (onset point → opaque point). The
  effect only looks right when that vector is nearly parallel to the
  camera's eye vector; accuracy degrades as the angle grows, breaking down
  completely at 90°. `set_linear_fallback(angle, onset, opaque)` defines a
  fallback: past `angle` degrees between fog vector and eye vector,
  `adjust_to_camera()` switches to plain camera-relative onset/opaque
  distances instead of the geometric projection. Default fallback cosine
  is `-1.0` (i.e. fallback essentially never triggers unless explicitly
  configured).
- `adjust_to_camera(camera_transform)` — called by the cull traverser each
  frame for linear-mode fog — computes the node's net transform relative
  to the camera, projects the onset/opaque points into camera-forward
  distance, and picks the fallback or geometric result per the check
  above. Results land in `_transformed_onset`/`_transformed_opaque`,
  fetched via `get_linear_range()`. If the Fog node has no parents
  (`get_num_parents() == 0`), it skips the NodePath-relative computation
  entirely and treats the points as already camera-relative.
- `xform()` transforms `_linear_onset_point`/`_linear_opaque_point` by the
  given matrix (unlike most PandaNode subclasses, which no-op `xform()`)
  — fog geometry is meaningful in object space and must follow flattening.

## API

| Method | Notes |
|---|---|
| `Fog(name)` | Constructor |
| `get_mode()` / `set_mode(Mode)` | `M_linear` \| `M_exponential` \| `M_exponential_squared` |
| `get_color()` / `set_color(r,g,b)` / `set_color(LColor)` | Alpha component unused |
| `set_linear_range(onset, opaque)` | Sets onset/opaque points along local forward axis; forces `M_linear` |
| `get_linear_onset_point()` / `set_linear_onset_point(...)` | Explicit 3-D onset point |
| `get_linear_opaque_point()` / `set_linear_opaque_point(...)` | Explicit 3-D opaque point |
| `set_linear_fallback(angle_deg, onset, opaque)` | Camera-relative fallback for steep fog-vector angles |
| `get_exp_density()` / `set_exp_density(float)` | `[0,1]`; forces `M_exponential` if mode was `M_linear` |
| `adjust_to_camera(TransformState*)` | Internal — called by cull traverser each frame |
| `get_linear_range(onset&, opaque&)` | Fetch post-`adjust_to_camera` transformed distances |

## Usage

```cpp
PT(Fog) fog = new Fog("scene_fog");
fog->set_color(0.5, 0.5, 0.6);
fog->set_linear_range(10.0, 100.0);
render.set_fog(fog);   // applies a FogAttrib referencing this node
```

## See also

- [FogAttrib](FogAttrib.md) — the RenderAttrib that activates a `Fog` node
- [Light](Light.md) — analogous graph-relative-effect pattern
