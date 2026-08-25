# Trackball

**Source:** `panda/src/tform/trackball.h` / `.cxx`
**Inherits:** [MouseInterfaceNode](MouseInterfaceNode.md)

`Trackball` is a "TFormer" that behaves like Performer's trackball mode: it
accumulates mouse-drag deltas into a rotation + translation, exposed as a
`transform` ([TransformState](../pgraph/TransformState.md)) output wire for
a [Transform2SG](Transform2SG.md) parented below it to apply to actual scene
graph geometry (or, with `set_invert(true)`, to a camera — making the world
appear to spin around a fixed viewer instead).

## Behavior notes

- **Mouse-button-to-action mapping is decoded from raw button state into a
  3-bit mask (`B1_MASK`/`B2_MASK`/`B3_MASK`) each frame, with Mac-style
  chording built in.** In `do_transmit_data()`: plain mouse button one is
  `B1_MASK`; button one **plus** alt is remapped to `B2_MASK` (and
  additionally to `B3_MASK` if meta/control is *also* held); button one
  plus meta/control (without alt) is remapped to `B3_MASK`; mouse buttons
  two and three set their own bits directly and combine normally. This
  keying-in of alt/meta/control only happens at all if
  `trackball_use_alt_keys` (config var, default `true`) caused those
  keyboard buttons to be `watch_button()`ed in the constructor.
- **A drag is only applied while the *same* button combination persists
  across two consecutive frames.** `do_transmit_data()` compares
  `this_button == _last_button` before calling `apply()` — the very first
  frame a new combination is pressed, the delta is computed but discarded
  (it only updates `_last_button`/`_lastx`/`_lasty`), preventing a
  spurious large jump on button-down.
- **Deltas are computed from `pixel_xy` (pixel coordinates), not the
  normalized `xy` wire** — `apply()`'s `x`/`y` are `this_x - _lastx` in
  raw pixel units, meaning the practical drag sensitivity implicitly scales
  with `_rotscale`/`_fwdscale` against window resolution rather than
  against a resolution-independent range.
- **`apply()`'s per-button-combination behavior is remapped again by
  `_control_mode`**, but only when the *decoded* button is exactly
  `B1_MASK` — `CM_truck` leaves it as-is, `CM_pan` remaps to `B2_MASK`,
  `CM_dolly` to `B3_MASK`, `CM_roll` to `B2_MASK|B3_MASK`; combinations
  that decoded to something other than plain `B1_MASK` (e.g. already
  `B2_MASK` from a real button two) are **not** affected by
  `_control_mode` at all.
- **Translation and rotation directions are coordinate-system-aware via
  `_cs`** (`LVector3::right(_cs)`, `up(_cs)`, `forward(_cs)`) — defaults to
  `get_default_coordinate_system()` but can be overridden with
  `set_coordinate_system()` so the same button semantics feel consistent
  regardless of the engine's configured handedness/up-axis.
- **`_rel_to` changes what space rotation/translation accumulate in, via
  an extract/recompute round-trip, not a one-time reparent.** Every call to
  `apply()` when a button is held first calls `reextract()` (which
  re-derives `_rotation`/`_translation` from `_orig` composed against
  `_rel_to`'s current transform), and every mutation ends with
  `recompute()` (which re-composes `_orig` from `_rotation`/`_translation`
  and *then* multiplies by `_rel_to`'s transform again) — so if `_rel_to`
  itself is moving (e.g. it's a vehicle node), the trackball's accumulated
  offset is continuously being reinterpreted relative to that moving frame
  each frame, not fixed at the moment `set_rel_to()` was called.
- **`set_invert()` doesn't just negate a flag consumed later — it decides
  which of `_mat` (world transform for scene parenting) or its inverse
  (camera transform) is what actually gets sent down the `transform` output
  wire.** `get_mat()` always returns `_orig` (uninverted); `get_trans_mat()`
  returns whatever was actually computed into `_mat` — these two can differ
  whenever `_invert` is true.

## API

### Construction / reset
| Signature | Notes |
|---|---|
| `explicit Trackball(const std::string &name)` | Watches mouse buttons 1-3 (and alt/meta/control if `trackball_use_alt_keys`) |
| `void reset()` | Resets rotation/translation/`_orig`/`_mat` to identity |

### Translation / rotation
| Signature | Notes |
|---|---|
| `get/set_pos`, `get_x/y/z`, `set_x/y/z` | Offset from the rotation origin |
| `get_hpr`, `get_h/p/r`, `set_hpr`, `set_h/p/r` | Derived via `decompose_matrix()`/`compose_matrix()` from `_rotation` each call |
| `void reset_origin_here()` | Moves the rotation origin to the current translation, zeroing translation |
| `void move_origin(x, y, z)` / `LPoint3 get_origin() const` / `void set_origin(...)` | |

### Speed / mode / space
| Signature | Notes |
|---|---|
| `get/set_forward_scale(PN_stdfloat)` | Scales dolly/truck sensitivity |
| `get/set_control_mode(ControlMode)` | `CM_default`, `CM_truck`, `CM_pan`, `CM_dolly`, `CM_roll` — see behavior notes |
| `get/set_invert(bool)` | World-transform vs. inverse-for-camera output |
| `get/set_rel_to(const NodePath &)` | Reference frame for accumulation, re-derived every frame |
| `get/set_coordinate_system(CoordinateSystem)` | Defaults to the engine default |
| `void set_mat(const LMatrix4 &)` / `const LMatrix4 &get_mat() const` / `get_trans_mat() const` | See invert caveat above |

### DataNode override
| Signature | Notes |
|---|---|
| `virtual void do_transmit_data(...)` | Input: `pixel_xy`. Output: `transform` |

## Usage

```cpp
NodePath mak_np = data_root.attach_new_node(new MouseAndKeyboard(win, 0, "mak"));
PT(Trackball) trackball = new Trackball("trackball");
NodePath tb_np = mak_np.attach_new_node(trackball);

trackball->set_invert(true);  // orbit the camera, not the scene

PT(Transform2SG) xform = new Transform2SG("cam_xform");
xform->set_node(camera.node());
tb_np.attach_new_node(xform);
```

## See also

[MouseInterfaceNode](MouseInterfaceNode.md) (base class — `watch_button`
mechanism) · [Transform2SG](Transform2SG.md) (typical downstream
consumer) · [DriveInterface](DriveInterface.md) (alternative
horizontal-plane TFormer)
