# MouseWatcher

**Source:** `panda/src/tform/mouseWatcher.h` / `.I` / `.cxx`
**Inherits:** `DataNode`, [MouseWatcherBase](MouseWatcherBase.md)

`MouseWatcher` is the central [data graph](../dgraph/README.md) node for
UI-style mouse/keyboard interaction: it tracks the current mouse position,
tests it against every [MouseWatcherRegion](MouseWatcherRegion.md) it (or
its attached [MouseWatcherGroup](MouseWatcherGroup.md)s) hold, fires the
region callback hooks and pattern-based Panda events, can suppress mouse or
keyboard data from propagating further down the graph, can drive a
software mouse cursor node, can log a mouse trail for gesture recognition,
and can auto-release held buttons after a period of inactivity.

## Behavior notes

- **"Entered" vs. "within" are two different, overlapping sets — unless
  `_enter_multiple` is set, in which case "within" implies "entered" too.**
  `set_current_regions()` computes the full set of regions the mouse
  geometrically overlaps (`get_over_regions()`, gathered from both direct
  `_regions` and every attached group) and diffs it against last frame's set
  to fire the internal `within_region()`/`without_region()` wrappers (in
  `mouseWatcher.I`) for *every* region in the overlap, nested or not — each
  of which throws `_within_pattern`/`_without_pattern` and calls the
  region's `within_region()`/`without_region()` hook. Separately, it picks
  exactly one "preferred" region via `get_preferred_region()` (highest
  `_sort`, then smallest area — see
  [MouseWatcherRegion::operator<](MouseWatcherRegion.md)) and fires
  `enter_region()`/`exit_region()` (throwing `_enter_pattern`/
  `_leave_pattern`) for transitions of that single region — *unless*
  `_enter_multiple` is true, in which case the `within_region()`/
  `without_region()` wrappers *also* call `enter_region()`/`exit_region()`
  for every overlapped region, not just the preferred one, so "entered"
  degenerates to "within" for all of them.
- **A button held down locks the preferred region for the duration of the
  press**, even if the mouse leaves it. `set_current_regions()` refuses to
  update `_preferred_region` while `_button_down` is true unless the new
  preferred region equals `_preferred_button_down_region` — so dragging the
  mouse off a button while holding it does not fire `exit_region()`/
  `enter_region()` on other regions until the button is released.
  `release()` then explicitly passes `param.set_outside(true)` if the mouse
  ended up somewhere other than the down-region.
- **Keyboard events always reach the preferred region, then optionally
  everyone else.** `press()`/`release()`/`keystroke()`/`candidate()` for
  non-mouse buttons unconditionally call the hook on `_preferred_region`
  first (regardless of that region's `get_keyboard()` flag), then broadcast
  to every other region with `get_keyboard()` set — *unless* the preferred
  region's `SF_other_button` suppress flag was triggered
  (`consider_keyboard_suppress()`), in which case the broadcast to "other"
  regions is skipped for the rest of that frame.
- **Mouse-button suppression and keyboard-button suppression are computed
  and applied independently, per frame, and reset every frame.**
  `_internal_suppress` (derived from `_preferred_region`'s
  `get_suppress_flags()`) and `_external_suppress` (set only via
  `consider_keyboard_suppress()` during keyboard dispatch) are both zeroed
  at the top of `do_transmit_data()`; their union gates whether
  `_button_events` (`SF_mouse_button`/`SF_other_button`) and `xy`/`pixel_xy`
  (`SF_mouse_position`) are passed to the output wires that frame — a `T_up`
  event is *never* suppressed regardless of these flags, to avoid a
  "stuck button" from a region losing interest mid-press.
- **`_implicit_click` simulates mouse button one on enter/exit**, not on
  hover — turning it on makes `enter_region()` synthesize a `press()` with
  `MouseButton::one()` and `exit_region()` synthesize the matching
  `release()`, useful for gaze/touch-style "hovering activates" UIs without
  wiring real button events.
- **The DisplayRegion constraint remembers which region "owns" an
  in-progress drag.** `constrain_display_region()` — used when
  `set_display_region()` narrows mouse tracking to one `DisplayRegion` —
  locks onto whichever `DisplayRegion` the button went down over
  (`_button_down_display_region`) and keeps rescaling mouse coordinates
  relative to *that* region until the button is released, even if the
  raw window-space mouse position wanders outside it; for a stereo
  `DisplayRegion` it recurses into both the left- and right-eye sub-regions.
- **The inactivity timeout is a 4-state machine, not a simple bool,** to
  avoid firing the timeout event or the "resume" logic more than once per
  transition: `IS_active` → (`elapsed > timeout`) → `IS_active_to_inactive`
  → (processed: release all held buttons, throw the timeout event) →
  `IS_inactive`; any activity while `IS_inactive` moves to
  `IS_inactive_to_active` → (processed: re-press all previously-held
  buttons) → `IS_active`. `note_activity()` only *cancels* a pending
  transition (`IS_active_to_inactive` → `IS_active`) or starts a resume
  (`IS_inactive` → `IS_inactive_to_active`); the actual button
  release/re-press happens once, in `do_transmit_data()`, not inside
  `note_activity()` itself.
- **The mouse trail log (`get_trail_node()`) is driven by `pointer_events`,
  a separate input wire from `xy`/`pixel_xy`**, and only accumulates while
  `_trail_log_duration > 0` — logging pointer events requires that the
  upstream `GraphicsWindowInputDevice` generate them *and* a nonzero
  duration be set via `set_trail_log_duration()`; otherwise the trail log
  input is read but immediately discarded each frame.
- **`is_button_down()` and `get_over_region()` don't do what their names
  alone suggest.** `is_button_down()` checks `_inactivity_state !=
  IS_inactive` in addition to the raw bit in `_current_buttons_down` — while
  the inactivity system considers all buttons logically released
  (`IS_inactive`), this query reports `false` even though the physical key
  may still be held and `_current_buttons_down`'s bit is still set.
  `get_over_region()` (no-arg overload) doesn't test any live geometry at
  all — it simply returns the cached `_preferred_region`, i.e. "the region
  the mouse was determined to be over as of the last `do_transmit_data()`,"
  not a fresh hit-test.
- **Software mouse-cursor geometry is positioned in `set_mouse()`, in the
  node's own transform space** — `_geometry->set_transform(TransformState
  ::make_pos(xy[0], 0, xy[1]))` — with no scaling applied, so the geometry's
  own units must already match the normalized `(-1..1)` mouse-coordinate
  space it's being placed in (typically achieved by parenting flat 2-D
  geometry under `render2d`).

## API

### Mouse state (PUBLISHED)
| Signature | Notes |
|---|---|
| `bool has_mouse() const` / `bool is_mouse_open() const` | Whether the mouse is currently within the window/frame |
| `const LPoint2 &get_mouse() const` / `get_mouse_x/y() const` | Normalized `(-1..1)` position |
| `void set_frame(...)` / `const LVecBase4 &get_frame() const` | Restricts the "window" this watcher considers active, default `(-1,1,-1,1)` |
| `bool is_over_region(...) const` / `MouseWatcherRegion *get_over_region(...) const` | Query without waiting for next frame's dispatch |
| `bool is_button_down(ButtonHandle) const` | |

### Event patterns (PUBLISHED, inline)
| Signature | Notes |
|---|---|
| `set/get_button_down_pattern`, `_button_up_pattern`, `_button_repeat_pattern` | `%r`=region name, `%b`=button name substitution |
| `set/get_enter_pattern`, `_leave_pattern` | Fired on preferred-region enter/exit |
| `set/get_within_pattern`, `_without_pattern` | Fired for every overlapped region (not just the preferred one); see behavior notes |

### Geometry / handlers / display region (PUBLISHED, inline)
| Signature | Notes |
|---|---|
| `set/get/has/clear_geometry(PandaNode *)` | Software mouse cursor node |
| `set/get_extra_handler(EventHandler *)` | Secondary `EventHandler` that also receives pattern events |
| `set/get_modifier_buttons(const ModifierButtons &)` | |
| `set/clear/get/has_display_region(DisplayRegion *)` | Constrain mouse tracking to one region (see behavior notes) |

### Groups
| Signature | Notes |
|---|---|
| `bool add_group(MouseWatcherGroup *)` / `bool remove_group(...)` | See [MouseWatcherGroup](MouseWatcherGroup.md) |
| `bool replace_group(MouseWatcherGroup *old_group, *new_group)` | Atomic swap that preserves common regions' "current" status |
| `int get_num_groups() const` / `MouseWatcherGroup *get_group(int) const` | |

### Inactivity timeout (PUBLISHED, inline)
| Signature | Notes |
|---|---|
| `set/has/get/clear_inactivity_timeout(double)` | Seconds of no activity before auto-releasing held buttons |
| `set/get_inactivity_timeout_event(const std::string &)` | Event thrown on timeout, default `"inactivity_timeout"` |
| `void note_activity()` | Manually reset the timer (e.g. for joystick input the watcher doesn't see) |

### Mouse trail (PUBLISHED)
| Signature | Notes |
|---|---|
| `void set_trail_log_duration(double)` | 0 disables logging and clears the log |
| `CPT(PointerEventList) get_trail_log() const` / `size_t num_trail_recent() const` | |
| `PT(GeomNode) get_trail_node()` / `void clear_trail_node()` | Debug visualization geometry, continually updated while held |

### DataNode override
| Signature | Notes |
|---|---|
| `virtual void do_transmit_data(DataGraphTraverser *, const DataNodeTransmit &input, DataNodeTransmit &output)` | Inputs: `pixel_xy`, `pixel_size`, `xy`, `button_events`, `pointer_events`. Outputs: `pixel_xy`, `pixel_size`, `xy`, `button_events` |

## Usage

```cpp
NodePath mak_np = data_root.attach_new_node(new MouseAndKeyboard(win, 0, "mak"));
PT(MouseWatcher) watcher = new MouseWatcher("watcher");
NodePath watcher_np = mak_np.attach_new_node(watcher);

PT(MouseWatcherRegion) region =
  new MouseWatcherRegion("ok_button", LVecBase4(-0.3, 0.3, -0.1, 0.1));
region->set_suppress_flags(MouseWatcherRegion::SF_mouse_button);
watcher->add_region(region);

watcher->set_enter_pattern("region-enter-%r");
watcher->set_button_down_pattern("region-press-%r-%b");
```

## See also

[MouseWatcherBase](MouseWatcherBase.md) (region storage this class
inherits) · [MouseWatcherRegion](MouseWatcherRegion.md),
[MouseWatcherParameter](MouseWatcherParameter.md) (what gets tested and
passed) · [MouseWatcherGroup](MouseWatcherGroup.md) (bulk region
attach/detach) · [DataNode](../dgraph/DataNode.md) (base wire protocol) ·
[display/MouseAndKeyboard](../display/MouseAndKeyboard.md) (typical
upstream data-graph parent)
