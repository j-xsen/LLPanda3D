# BillboardEffect

**Source:** `panda/src/pgraph/billboardEffect.h` (+ `.I`, `.cxx`)
**Inherits:** RenderEffect

A `RenderEffect` that automatically rotates the node it's attached to so it
faces the camera (or an arbitrary other node), each frame, at cull time.
Used for sprites, particle billboards, and similar always-facing geometry.

## Behavior notes

- `make()` is the general constructor; `make_axis()`, `make_point_eye()`,
  and `make_point_world()` are convenience wrappers for the three common
  presets (see API table).
- **Axial vs. point rotation:** `axial_rotate=true` rotates only around the
  `up_vector` axis (`heads_up()` — good for trees/signs that should stay
  upright); `axial_rotate=false` rotates freely in 3D to face the target
  (`look_at()` — good for particle sprites).
- **Eye-relative vs. world-relative:** `eye_relative=true` interprets
  `up_vector` relative to the camera and looks toward the plane
  perpendicular to the camera's forward axis (not straight at the camera
  point) — this is the standard "sprite" mode. `eye_relative=false`
  interprets `up_vector` in world space and looks straight at
  `look_at_point` (relative to `look_at`, or the camera if `look_at` is
  empty).
- `offset`/`fixed_depth`: after rotating, the geometry is slid toward the
  camera by `offset` units along the view direction; if `fixed_depth` is
  true the geometry is instead pinned at a constant apparent depth
  (`offset` is then interpreted as `-depth`). Useful to keep billboards
  from being clipped into nearby geometry.
- `look_at` empty means "face the current camera" (camera-dependent,
  resolved at cull time via `CullTraverser::get_camera_transform()`);
  non-empty means "face this specific NodePath", which makes the effect
  have `has_adjust_transform() == true` — i.e. it affects the node's
  *net* transform outside of cull time too (e.g. for bounding volume
  computation), not just what's drawn.
- `safe_to_transform()` returns `false` — a billboarded node's local
  transform shouldn't be baked/flattened away, since the effect depends on
  its runtime relationship to the camera.
- `prepare_flatten_transform()` strips rotation (sets HPR to zero) from any
  transform about to be flattened through this node — flattening must not
  bake in a rotation that the billboard effect is responsible for undoing
  every frame.
- An "off" `BillboardEffect` (`is_off() == true`) means no billboarding;
  this only shows up as an implicit result of `NodePath::get_rel_state()`
  and is not something constructed directly.

## API

| Method | Notes |
|---|---|
| `static CPT(RenderEffect) make(const LVector3 &up_vector, bool eye_relative, bool axial_rotate, PN_stdfloat offset, const NodePath &look_at, const LPoint3 &look_at_point, bool fixed_depth = false)` | General constructor |
| `static CPT(RenderEffect) make_axis()` | Axis-rotating, up-vector `(0,0,1)`, world-relative |
| `static CPT(RenderEffect) make_point_eye()` | Point-rotating, eye-relative — standard sprite billboard |
| `static CPT(RenderEffect) make_point_world()` | Point-rotating, world-relative |
| `bool is_off() const` | True if this is the no-op "off" effect |
| `const LVector3 &get_up_vector() const` | |
| `bool get_eye_relative() const` | |
| `bool get_axial_rotate() const` | |
| `bool get_fixed_depth() const` | |
| `PN_stdfloat get_offset() const` | |
| `const NodePath &get_look_at() const` | Empty means "face the camera" |
| `const LPoint3 &get_look_at_point() const` | Point relative to `look_at`, default origin |

## Usage

```cpp
// Standard camera-facing sprite billboard.
node_path.set_effect(BillboardEffect::make_point_eye());

// Upright tree/sign that only rotates around Z to face the camera.
node_path.set_effect(BillboardEffect::make_axis());
```

## See also

- [RenderEffect](RenderEffect.md) — base class
- [../pgraph/README.md](README.md) — cull pipeline / state pipeline overview
