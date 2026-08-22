# PGSliderBar

**Source:** `panda/src/pgui/pgSliderBar.h` / `.I` / `.cxx` (+ `pgSliderBarNotify.h/.cxx`)
**Inherits:** [PGItem](PGItem.md), `PGButtonNotify` → `PandaNode`

A draggable slider / scroll bar: a trough the thumb moves along, optionally
with left/right (or up/down) step buttons. Backs both DirectSlider and
DirectScrollBar in Python, and is used internally by [PGScrollFrame](PGScrollFrame.md)
for its two scroll bars.

## Behavior notes

- **Value is stored internally as a normalized ratio `[0, 1]`** (`_ratio`);
  `set_value()`/`get_value()` are thin wrappers that map to/from
  `[min_value, max_value]`. If you only care about position (e.g. a scroll
  bar), use `set_ratio()`/`get_ratio()` directly and ignore the value range.
  Setting `min_value == max_value` is asserted against.
- **`set_ratio()`/`set_value()` are ignored while the user is manipulating the
  slider** (`is_button_down()` true — dragging the thumb, holding a scroll
  button, or holding the mouse down on the trough). Programmatic updates during
  a drag are silently dropped rather than fighting the user's input; use
  `internal_set_ratio()` (private) semantics are not exposed for that reason.
- **`is_button_down()`** covers three distinct interaction modes: dragging the
  thumb (`_dragging`), holding the mouse down directly on the trough
  (`_mouse_button_page`), and holding a left/right step button
  (`_scroll_button_held != nullptr`).
- **`setup_scroll_bar()` builds three `PGButton` children** (thumb, left,
  right) and enables `resize_thumb` (thumb width visually represents
  `page_size`) and `manage_pieces`. **`setup_slider()`** builds only a thumb
  button (no step buttons), disables `resize_thumb`, and uses `visible_scale`
  on the frame style to draw a thinner track than the clickable frame.
- **`axis` must be exactly one of the four screen-axis unit vectors**
  `(1,0,0)`, `(-1,0,0)`, `(0,0,1)`, `(0,0,-1)` — anything else is
  "indeterminate behavior" per the header comment. `setup_scroll_bar`/
  `setup_slider` set this correctly for you based on the `vertical` flag; only
  set it directly if you're building a fully custom slider.
- **`manage_pieces`** auto-positions/sizes the thumb and left/right buttons
  whenever the slider's frame changes (`remanage()`), based on whichever axis
  ("X-dominant" vs "Y-dominant", i.e. `|axis.x| > |axis.y+axis.z|`) the slider
  runs along.
- **Clicking directly on the trough (not the thumb) pages** by `page_size -
  scroll_size` toward the click point (leaving a `scroll_size` margin so it
  doesn't overshoot past the click), and if the thumb reaches exactly the
  target ratio, dragging begins automatically so continued mouse movement
  keeps tracking the cursor (`advance_page()` → `begin_drag()`).
- **Holding a scroll button or the trough auto-repeats**, gated by
  `scroll-initial-delay` / `scroll-continued-delay` config variables (see
  `config_pgui`, defaults 0.3s / 0.1s).

## Notify interface — `PGSliderBarNotify`

Extends [PGItemNotify](PGItem.md#notify-interface--pgitemnotify):

```cpp
class PGSliderBarNotify : public PGItemNotify {
protected:
  virtual void slider_bar_adjust(PGSliderBar *slider_bar);      // value changed
  virtual void slider_bar_set_range(PGSliderBar *slider_bar);   // range/min/max changed
};
```

Attach with `set_notify(PGSliderBarNotify*)`. [PGScrollFrame](PGScrollFrame.md)
uses this internally to react to its horizontal/vertical scroll bars moving.
`PGSliderBar` itself is also a `PGButtonNotify`, internally watching its own
thumb/left/right `PGButton`s via `item_press`/`item_release`/`item_move`.

## API

### Setup
| Signature | Notes |
|---|---|
| `void setup_scroll_bar(bool vertical, PN_stdfloat length, width, bevel)` | Trough + thumb + left/right step buttons |
| `void setup_slider(bool vertical, PN_stdfloat length, width, bevel)` | Trough + thumb only, thinner visible track |

### Axis
| Signature | Notes |
|---|---|
| `void set_axis(const LVector3&)` / `const LVector3 &get_axis() const` | Must be a unit screen axis — see "Behavior notes" |

### Range & value
| Signature | Notes |
|---|---|
| `void set_range(PN_stdfloat min, PN_stdfloat max)` | `min != max` required |
| `PN_stdfloat get_min_value() const` / `get_max_value() const` | |
| `void set_scroll_size(PN_stdfloat)` / `get_scroll_size() const` | Step amount for left/right buttons |
| `void set_page_size(PN_stdfloat)` / `get_page_size() const` | Trough-click jump amount; also thumb size if `resize_thumb` |
| `void set_value(PN_stdfloat)` / `get_value() const` | In `[min_value, max_value]`; no-op while `is_button_down()` |
| `void set_ratio(PN_stdfloat)` / `get_ratio() const` | In `[0, 1]`; no-op while `is_button_down()` |
| `bool is_button_down() const` | True while user is dragging/holding a button/holding the trough |

### Pieces
| Signature | Notes |
|---|---|
| `void set_resize_thumb(bool)` / `get_resize_thumb() const` | Thumb width tracks `page_size` |
| `void set_manage_pieces(bool)` / `get_manage_pieces() const` | Auto-layout of thumb/step buttons |
| `void set_thumb_button(PGButton*)` / `clear_thumb_button()` / `get_thumb_button() const` | Caller must parent the button to the slider node |
| `void set_left_button(PGButton*)` / `clear_left_button()` / `get_left_button() const` | Optional |
| `void set_right_button(PGButton*)` / `clear_right_button()` / `get_right_button() const` | Optional |

### Events / lifecycle
| Signature | Notes |
|---|---|
| `static std::string get_adjust_prefix()` → `"adjust-"` | |
| `std::string get_adjust_event() const` | `"adjust-" + get_id()` |
| `void remanage()` | Force-reposition/size pieces now |
| `void recompute()` | Force-recompute thumb travel range now |
| `void set_notify(PGSliderBarNotify*)` / `PGSliderBarNotify *get_notify() const` | |

Also overrides `set_active(bool)` (propagates to thumb/left/right buttons).

## Events

| Event | Fires when | Params |
|---|---|---|
| `adjust-<id>` | value changed, by user drag/click/scroll or by `set_value()`/`set_ratio()` | *(none — plain `throw_event`)* |

Plus base [PGItem events](PGItem.md#events).

## Usage

```cpp
PT(PGSliderBar) slider = new PGSliderBar("volume");
slider->setup_slider(false, /*length*/ 2.0f, /*width*/ 0.3f, /*bevel*/ 0.05f);
slider->set_range(0.0f, 1.0f);
slider->set_value(0.5f);
top_np.attach_new_node(slider);

accept(slider->get_adjust_event(), [slider](const Event *) {
  float v = slider->get_value();
});
```

## See also

[PGItem.md](PGItem.md) · [PGButton.md](PGButton.md) (thumb/step buttons) ·
[PGScrollFrame.md](PGScrollFrame.md) (uses two of these as scroll bars) ·
[README.md](README.md)
