# PGScrollFrame

**Source:** `panda/src/pgui/pgScrollFrame.h` / `.I` / `.cxx`
**Inherits:** [PGVirtualFrame](PGVirtualFrame.md) → [PGItem](PGItem.md) → `PandaNode`; also `PGSliderBarNotify`

The familiar "scrolled panel" widget: a [PGVirtualFrame](PGVirtualFrame.md)
(clip window + canvas) plus two [PGSliderBar](PGSliderBar.md)s that drive the
canvas's scroll position, with automatic layout and optional auto-hide of
scroll bars that aren't needed. This is what backs DirectScrolledFrame.

## Behavior notes

- **`setup()` builds everything**: background frame, clip frame, both scroll
  bars (as child `PGSliderBar`s wired via `set_horizontal_slider`/
  `set_vertical_slider`), and enables `manage_pieces` + `auto_hide`. Calling it
  again first removes any scroll bars created by a previous call.
  `virtual_frame` is the size of the content ("the size of the large, virtual
  canvas which we can see only a portion of at any given time").
- **`virtual_frame` vs. `clip_frame` vs. actual `frame`**: `frame` is the whole
  widget's clickable/visual bounds; `clip_frame` (inherited, computed from
  `frame` minus border, minus space for visible scroll bars) is the visible
  window; `virtual_frame` is the logical size of the scrollable content. If no
  virtual frame is set, `get_virtual_frame()` falls back to the clip frame
  (i.e. nothing to scroll).
- **Recomputation is split into three lazily-flagged passes**, each run once
  per frame from `cull_callback()` in this order: `remanage()` (scroll bar
  position/size, if `manage_pieces` and `_needs_remanage`) →
  `recompute_clip()` (clip frame + slider page sizes, if
  `_needs_recompute_clip`) → `recompute_canvas()` (canvas scroll transform, if
  the atomic `_canvas_computed` flag is clear). Almost every setter just flips
  one of these flags rather than doing work immediately; call `recompute()`
  (calls `recompute_clip()` + `recompute_canvas()`) or `remanage()` directly
  only if you need results before the next `cull_callback()`.
- **`auto_hide` implies `manage_pieces`** — setting `auto_hide(true)` forces
  `manage_pieces` on too, since hiding/showing bars requires managed layout.
  With `auto_hide` on, a scroll bar is hidden (and its ratio reset to 0,
  freeing up its width for the other bar and the clip area) whenever the
  virtual frame already fits within the available clip area on that axis.
- **`item_frame_changed`/`item_transform_changed`/`item_draw_mask_changed`**
  are the `PGItemNotify` overrides used to detect when a scroll bar's own size
  changed (which affects how much clip-area space it consumes) —
  `PGScrollFrame` calls `slider->set_notify(this)` on both bars automatically
  when you assign them.
- **`slider_bar_adjust()`** (the `PGSliderBarNotify` override) doesn't
  recompute synchronously — it just clears the `_canvas_computed` atomic flag
  so the next `cull_callback()` picks up the new scroll position. This makes
  repeated scroll-bar drags within one frame cheap.
- **The canvas is translated, never scaled/clipped by resizing content** — the
  scroll offset (`interpolate_canvas()`) linearly maps each slider's `[0,1]`
  ratio to the range needed to slide the virtual frame's edge to the clip
  frame's edge on that axis.

## API

### Setup
| Signature | Notes |
|---|---|
| `void setup(PN_stdfloat width, height, left, right, bottom, top, slider_width, bevel)` | Builds background, clip frame, both scroll bars. `left/right/bottom/top` define the virtual (content) frame |

### Virtual frame
| Signature | Notes |
|---|---|
| `void set_virtual_frame(left, right, bottom, top)` / `set_virtual_frame(const LVecBase4&)` | Size of the scrollable content |
| `const LVecBase4 &get_virtual_frame() const` | Falls back to `get_clip_frame()` if unset |
| `bool has_virtual_frame() const` / `void clear_virtual_frame()` | |

### Layout flags
| Signature | Notes |
|---|---|
| `void set_manage_pieces(bool)` / `get_manage_pieces() const` | Auto position/size the two scroll bars |
| `void set_auto_hide(bool)` / `get_auto_hide() const` | Hide unneeded scroll bars; forces `manage_pieces` true |

### Scroll bars
| Signature | Notes |
|---|---|
| `void set_horizontal_slider(PGSliderBar*)` / `clear_horizontal_slider()` / `get_horizontal_slider() const` | Caller must parent the slider to the frame node |
| `void set_vertical_slider(PGSliderBar*)` / `clear_vertical_slider()` / `get_vertical_slider() const` | Same |

### Recompute
| Signature | Notes |
|---|---|
| `void remanage()` | Force scroll-bar layout now |
| `void recompute()` | Force clip + canvas recompute now |

### Overridden hooks (from `PGItemNotify` / `PGSliderBarNotify`, protected)
`item_transform_changed`, `item_frame_changed`, `item_draw_mask_changed`,
`slider_bar_adjust` — internal wiring; not normally called or overridden by
application code.

## Events

Inherits [PGItem events](PGItem.md#events). Its child `PGSliderBar`s throw
their own `adjust-<slider_id>` events (see [PGSliderBar](PGSliderBar.md#events))
if you need to react to scroll position directly rather than polling
`get_canvas_transform()`.

## Usage

```cpp
PT(PGScrollFrame) frame = new PGScrollFrame("panel");
frame->setup(/*width*/4.0f, /*height*/3.0f,
             /*virtual left,right,bottom,top*/ 0.0f, 4.0f, -10.0f, 0.0f,
             /*slider_width*/ 0.3f, /*bevel*/ 0.05f);
top_np.attach_new_node(frame);

NodePath content = NodePath(frame->get_canvas_node()).attach_new_node("list");
// ... populate content, taller than the 3-unit clip height ...
// scroll bars appear automatically since virtual height (10) > clip height (~3)
```

## See also

[PGVirtualFrame.md](PGVirtualFrame.md) (base clipping mechanism) ·
[PGSliderBar.md](PGSliderBar.md) (the scroll bars) · [PGItem.md](PGItem.md) ·
[README.md](README.md)
