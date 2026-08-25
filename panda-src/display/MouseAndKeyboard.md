# MouseAndKeyboard

**Source:** `panda/src/display/mouseAndKeyboard.h` (+ `.cxx`, no `.I`)
**Inherits:** DataNode (external base, `panda/src/dgraph`) **Inherited by:** (none)

Reads one input device's mouse/keyboard state off a
[GraphicsWindow](GraphicsWindow.md) each frame and republishes it as outputs
on the **data graph** — the separate node graph (rooted at, e.g.,
`PandaFramework::get_data_root()`) that carries per-frame device input,
distinct from the 3-D/2-D scene graphs. A `DataNode` is a data-graph node
with typed named inputs/outputs, traversed once per frame by a
`DataGraphTraverser`; `MouseAndKeyboard` defines no inputs (it's a source
node) and five outputs (below). This is exactly the class
`PandaFramework::get_mouse(window)` creates one instance of per window (see
[../framework/PandaFramework.md](../framework/PandaFramework.md) and
[../framework/README.md](../framework/README.md)); downstream data-graph
nodes like `MouseWatcher` and `Trackball` (both referenced, undocumented,
in the `framework` module) consume its outputs.

## Behavior notes

- **Mouse and keyboard are bundled into one node deliberately** — per the
  class comment, because they interrelate (shift-constrained mouse drags,
  mouse clicks handled like key events). Keyboard events surface as a
  `ButtonEventList` output; mouse motion surfaces as both raw pixel
  coordinates and a normalized `[-1,1]` position; **nothing throws these as
  Panda `Event`s by itself** — an `EventThrower` data-graph node must be
  attached downstream of `MouseAndKeyboard`'s `button_events` output, or
  the button events are silently discarded (this is what
  `WindowFramework::enable_keyboard()`'s `ButtonThrower` is for, one level
  further down the data graph in the `framework` module).
- **The constructor resolves `device` (an integer index) to an
  `InputDevice*` via `GraphicsWindow::get_input_device(device)`** and
  immediately calls `enable_pointer_events()` on it. `do_transmit_data()`
  then C-style-casts that stored `InputDevice*` down to
  `GraphicsWindowInputDevice*` unconditionally — see
  [GraphicsWindowInputDevice.md](GraphicsWindowInputDevice.md). This is
  safe in practice only because `GraphicsWindow::get_input_device()` always
  returns a `GraphicsWindowInputDevice` for device index within a window,
  but it's not statically enforced by the type used to store `_device`
  (plain `InputDevice`).
- **`set_source()` can retarget the node to a different window/device
  after construction** — but its call to re-enable pointer events on the
  new device is commented out in the source (`//_device->enable_pointer_events();`),
  so retargeting does not automatically re-enable pointer event reporting
  on the new device; callers relying on `set_source()` should verify
  pointer events are still enabled if the new device needs it.
- **Pixel-to-normalized coordinate mapping** (in `do_transmit_data()`):
  pixel `xy` output is only populated `if (mdata._in_window)` — i.e. mouse
  position outputs go stale (not updated, and downstream nodes see the
  last-published value) whenever the pointer has left the window, rather
  than being explicitly cleared. The normalized `xy` output maps pixel
  `(0,0)` (top-left) to `(-1, 1)` and pixel `(w,h)` (bottom-right) to
  roughly `(1, -1)` — Y is flipped relative to pixel space to match
  Panda's render-coordinate convention (up is positive).
- **`pixel_size` output is only populated if the window currently reports a
  size** (`WindowProperties::has_size()`) — on a window that hasn't
  finished opening or doesn't report size, none of `pixel_xy`/`pixel_size`/
  `xy` are updated that frame, though `button_events`/`pointer_events` are
  unaffected by this (they're populated unconditionally whenever the
  device reports them).

## API

| Signature | Notes |
|---|---|
| `explicit MouseAndKeyboard(GraphicsWindow *window, int device, const std::string &name)` | `device` is an index into the window's input devices, resolved via `GraphicsWindow::get_input_device()`. Enables pointer events on that device immediately. |
| `void set_source(GraphicsWindow *window, int device)` | Retargets to a different window/device; does **not** re-enable pointer events (see behavior notes). |
| `virtual void do_transmit_data(DataGraphTraverser *trav, const DataNodeTransmit &input, DataNodeTransmit &output)` (protected) | The per-frame `DataNode` callback; not normally called directly. |

### Data-graph outputs (defined in the constructor, not public member accessors)

| Output name | Type | Populated when |
|---|---|---|
| `pixel_xy` | `EventStoreVec2` | Window has a known size and the pointer is currently inside it. |
| `pixel_size` | `EventStoreVec2` | Window has a known size. |
| `xy` | `EventStoreVec2` | Same conditions as `pixel_xy`; normalized to `[-1,1]`, Y-flipped. |
| `button_events` | `ButtonEventList` | Whenever the device reports `has_button_event()`. |
| `pointer_events` | `PointerEventList` | Whenever the device reports `has_pointer_event()`. |

## Usage

```cpp
#include "mouseAndKeyboard.h"

NodePath data_root("data");
PT(MouseAndKeyboard) mak = new MouseAndKeyboard(window, 0, "mouse");
NodePath mouse = data_root.attach_new_node(mak);

// Downstream, e.g. a MouseWatcher (panda/src/tform) filters/clips this,
// and a Trackball or ButtonThrower consumes its outputs.
```

## See also

- [GraphicsWindowInputDevice.md](GraphicsWindowInputDevice.md) — the actual device this node reads from via `GraphicsWindow::get_input_device()`.
- [GraphicsWindow.md](GraphicsWindow.md) — owns the input device list and current `WindowProperties`/size this node queries every frame.
- [../framework/PandaFramework.md](../framework/PandaFramework.md), [../framework/README.md](../framework/README.md) — `PandaFramework::get_mouse()` is the framework-level factory for one `MouseAndKeyboard` per window; `WindowFramework::get_mouse()`/`enable_keyboard()` build the `MouseWatcher`/`ButtonThrower` chain downstream of it.
- [../tform/README.md](../tform/README.md) — the `MouseWatcher`, `Trackball`, `DriveInterface`, `ButtonThrower`, etc. classes that consume this node's `xy`/`pixel_xy`/`button_events` outputs.
