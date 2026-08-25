# DriveInterface

**Source:** `panda/src/tform/driveInterface.h` / `.cxx`
**Inherits:** [MouseInterfaceNode](MouseInterfaceNode.md)

`DriveInterface` is a "TFormer" (transform-generating node) that turns mouse
position and/or arrow-key state into a horizontal-plane "drive a vehicle"
style movement, outputting a `transform` ([TransformState](../pgraph/TransformState.md))
and a `velocity` (`EventStoreVec3`) wire for something like a
[Transform2SG](Transform2SG.md) to apply downstream. Config defaults for
speed/dead-zone/ramp values come from `config_tform.h`
(`drive-forward-speed`, `drive-rotate-speed`, etc.).

## Behavior notes

- **Mouse control and keyboard (arrow key) control are mutually exclusive
  per frame, decided by whether mouse button one is down.** `apply()`
  branches on `any_button || _force_mouse`: if true, speed/rotation come
  from the mouse's position within vertical/horizontal dead zones around
  `_vertical_center`/`_horizontal_center`; if false, they come from the
  arrow-key ramp state instead. `watch_button(MouseButton::one())` in the
  constructor is what makes `is_down(MouseButton::one())` available to
  drive this branch (combined with `!_ignore_mouse`, from the caller in
  `do_transmit_data()`).
- **The dead zone is a linear ramp from its edge to the window edge, not a
  hard cutoff beyond it.** E.g. for the vertical/forward axis: below
  `dead_zone_top` nothing happens; above it, `throttle = (y -
  dead_zone_top) / (1.0 - dead_zone_top)` scales linearly from 0 at the
  dead-zone edge to 1 at `y = 1`, then `_speed = throttle *
  _forward_speed`. Backward motion mirrors this against
  `_reverse_speed` on the opposite side.
- **Keyboard ramping models inertia via a "most-recently-changed key wins"
  rule, per `KeyHeld`.** `KeyHeld::operator<` prefers whichever of a
  key-pair (up/down or left/right) is currently held; if both or neither
  are held, the one whose `_changed_time` is more recent wins. That winning
  key's `get_effect()` computes a 0..1 ramp based on elapsed time since it
  last toggled state, scaled against `ramp_up_time` (while held) or
  `ramp_down_time` (while released) — a ramp time of exactly `0.0` skips the
  ramp entirely and snaps to the target value.
- **The losing key's `_effect` is force-reset to 0 every frame it loses**,
  even if it was mid-ramp — e.g. if up-arrow is held and then down-arrow is
  also pressed, up wins (more recently changed) and `_down_arrow._effect =
  0.0f` is set unconditionally, discarding any ramp-down progress
  down-arrow might have had.
- **The `_hpr_quantize` static member (`0.001`) is declared but never
  referenced anywhere in `driveInterface.cxx`** — the comment above it in
  the header describes an intended use (filtering numerical-noise
  perturbations from the public `set_hpr()` interface) that the current
  implementation does not actually perform.
- **`set_force_roll()` is a documented no-op** — "This function is no
  longer used and does nothing. It will be removed soon" (per its own doc
  comment); calling it has no effect on state.
- **`force_dgraph()` manually re-drives a partial traversal below this
  node** to eliminate a one-frame latency after directly calling
  `set_pos()`/`set_hpr()`/`set_mat()` (e.g. after a collision response) —
  it builds a fresh `DataNodeTransmit` with just this node's outputs and
  calls `DataGraphTraverser::traverse_below()` + `collect_leftovers()`
  itself, rather than waiting for the next natural frame traversal to reach
  downstream consumers.
- **Vertical drift is corrected per coordinate system, not universally
  zeroed** — after computing a forward step via the current heading's
  rotation matrix, `apply()` zeroes `step[2]` for `CS_zup_*` systems or
  `step[1]` for `CS_yup_*` systems specifically, based on
  `get_default_coordinate_system()`, rather than always zeroing a fixed
  axis.

## API

### Speed / dead zone / ramp tuning (PUBLISHED, inline)
| Signature | Notes |
|---|---|
| `set/get_forward_speed`, `_reverse_speed`, `_rotate_speed` | Units/sec (or deg/sec) at full deflection |
| `set/get_vertical_dead_zone`, `_horizontal_dead_zone` | Fraction of window before motion begins |
| `set/get_vertical_ramp_up_time`, `_ramp_down_time`, `_horizontal_ramp_up_time`, `_ramp_down_time` | Keyboard-only inertia, seconds; `0` = instant |
| `get_speed() const` / `get_rot_speed() const` | Instantaneous computed values from the last `apply()` |

### Position / orientation (PUBLISHED, inline)
| Signature | Notes |
|---|---|
| `get/set_pos(...)`, `get/set_x/y/z(...)` | |
| `get/set_hpr(...)`, `get/set_h/p/r(...)` | |
| `void reset()` | Resets position/orientation to origin and clears all `KeyHeld` state |
| `void set_mat(const LMatrix4 &)` / `const LMatrix4 &get_mat()` | Decomposes/recomposes via `decompose_matrix()`/`compose_matrix()` |

### Misc (PUBLISHED, inline unless noted)
| Signature | Notes |
|---|---|
| `set/get_ignore_mouse(bool)` | Forces keyboard-only control even with button one held |
| `set/get_force_mouse(bool)` | Forces mouse control even without button one held |
| `set/get_stop_this_frame(bool)` | One-shot: zeroes this frame's distance/rotation, then clears itself |
| `void force_dgraph()` | See behavior notes |

### DataNode override
| Signature | Notes |
|---|---|
| `virtual void do_transmit_data(...)` | Inputs: `xy`, `button_events`. Outputs: `transform`, `velocity` |

## Usage

```cpp
NodePath mak_np = data_root.attach_new_node(new MouseAndKeyboard(win, 0, "mak"));
PT(DriveInterface) drive = new DriveInterface("drive");
NodePath drive_np = mak_np.attach_new_node(drive);

PT(Transform2SG) xform = new Transform2SG("xform");
xform->set_node(camera.node());
drive_np.attach_new_node(xform);
```

## See also

[MouseInterfaceNode](MouseInterfaceNode.md) (base class — `watch_button`
mechanism) · [Transform2SG](Transform2SG.md) (typical downstream consumer
of the `transform` output) · [Trackball](Trackball.md) (alternative
orbit-style TFormer)
