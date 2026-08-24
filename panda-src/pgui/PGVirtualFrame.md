# PGVirtualFrame

**Source:** `panda/src/pgui/pgVirtualFrame.h` / `.I` / `.cxx`
**Inherits:** [PGItem](PGItem.md) → `PandaNode`
**Inherited by:** [PGScrollFrame](PGScrollFrame.md)

A window onto an arbitrarily large "virtual canvas": anything parented under
`get_canvas_node()` is clipped to the `clip_frame` rectangle via a
`ScissorEffect`, and can be scrolled by changing `canvas_transform` (a plain
translate/scale/rotate on the canvas node). This is the low-level clipping
mechanism; [PGScrollFrame](PGScrollFrame.md) builds the familiar "scroll bars +
auto-hide + drag" behavior on top of it and is what is normally used
directly.

## Behavior notes

- **Two extra child nodes are always present**, created in the constructor:
  `canvas_parent` (holds the `ScissorEffect`) → `canvas_node` (translate freely
  via `canvas_transform`; this is where scrollable content is parented). Both are
  `ModelNode`s with `PT_local` preserve-transform, so copying/flattening the
  scene graph won't collapse their transforms into neighbors.
  `get_canvas_parent()` exists mainly for the copy/instance machinery; content
  should be parented under `get_canvas_node()`.
- **Clipping only takes effect if a clip frame has been set** — `has_clip_frame()`
  false (the default, until `set_clip_frame()`/`setup()` is called) means
  content is *not* clipped at all.
  `get_clip_frame()` falls back to the item's own `get_frame()` when no
  explicit clip frame is set.
  `set_clip_frame()` no-ops if the new value equals the current one (avoids
  regenerating the `ScissorEffect` needlessly).
- **`clip_frame_changed()` is a protected virtual hook**, empty in the base
  class — [PGScrollFrame](PGScrollFrame.md) overrides it to know when it needs
  to recompute scroll-bar page sizes.
- **Copying (`make_copy()` / scene-graph copy) correctly re-links the new
  copy's `canvas_node`/`canvas_parent`** via `r_copy_children()`'s use of the
  copy's `InstanceMap`, and reapplies the clip frame's `ScissorEffect` on the
  new parent node — no manual fix-up is needed after copying a
  `PGVirtualFrame` subtree.

## API

### Setup
| Signature | Notes |
|---|---|
| `void setup(PN_stdfloat width, PN_stdfloat height)` | Minimal default: bevel-out frame + clip frame inset by a small fixed bevel. Real usage is normally via `PGScrollFrame::setup()` instead |

### Clip frame
| Signature | Notes |
|---|---|
| `void set_clip_frame(PN_stdfloat left, right, bottom, top)` / `set_clip_frame(const LVecBase4&)` | The visible "window" rectangle, local coords |
| `const LVecBase4 &get_clip_frame() const` | Falls back to `get_frame()` if unset |
| `bool has_clip_frame() const` | |
| `void clear_clip_frame()` | Disables clipping entirely |

### Canvas
| Signature | Notes |
|---|---|
| `void set_canvas_transform(const TransformState*)` / `const TransformState *get_canvas_transform() const` | Scroll position/scale of the virtual canvas |
| `PandaNode *get_canvas_node() const` | Scrollable content is parented here |
| `PandaNode *get_canvas_parent() const` | Rarely needed directly |

### Hook
| Signature | Notes |
|---|---|
| `protected virtual void clip_frame_changed()` | Override to react to clip-frame resizes; no-op by default |

## Usage

```cpp
PT(PGVirtualFrame) frame = new PGVirtualFrame("panel");
frame->setup(4.0f, 3.0f);
top_np.attach_new_node(frame);

NodePath content = NodePath(frame->get_canvas_node()).attach_new_node("big_content");
// ... build content larger than the 4x3 clip window ...

frame->set_canvas_transform(TransformState::make_pos(LVector3(0, 0, -2)));  // scroll down
```

## See also

[PGItem.md](PGItem.md) · [PGScrollFrame.md](PGScrollFrame.md) (adds scroll bars
on top of this) · [README.md](README.md)
