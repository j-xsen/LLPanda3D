# PGButton

**Source:** `panda/src/pgui/pgButton.h` / `.cxx` (+ `pgButtonNotify.h/.cxx`)
**Inherits:** [PGItem](PGItem.md) → `PandaNode`

A clickable button. Tracks its own visual state (ready / depressed / rollover
/ inactive) automatically in response to mouse events and throws a `click-`
event when clicked normally (press-then-release while still over the button,
or a background-focus keyboard "click"). Used as the base building block for
[PGSliderBar](PGSliderBar.md)'s thumb/left/right buttons.

## Behavior notes

- **States are a fixed enum, not arbitrary ints:**
  `S_ready=0, S_depressed, S_rollover, S_inactive`. `setup()` builds default
  frame-style geometry for exactly these four states.
- **A click requires press *and* release over the button, with matching
  button handle in `_click_buttons`** (default: just `MouseButton::one()`,
  i.e. left mouse button — add more with `add_click_button()`). If the release
  happens outside the frame *and* it was a mouse button (or the item lacks
  focus), the button reverts to `S_ready` without clicking; otherwise it clicks
  even if the mouse drifted outside, as long as it's a keyboard-driven release
  while focused (supports keyboard-accessible buttons).
- **`setup(label, bevel)`** builds a complete default text-label button:
  generates label geometry once via `PGItem::get_text_node()`, sizes the frame
  to the text's bounding card plus fixed padding (`±0.4` X, `±0.15` Z), and
  creates ready/rollover/inactive bevel-out frame styles plus a bevel-in
  depressed style with the depressed content visually offset by `(0.05, 0,
  -0.05)` to read as "pressed in."
- **`setup(ready, depressed, rollover, inactive)`** instead uses four
  caller-supplied `NodePath`s as the per-state geometry (no frame style is
  generated) and sizes the frame from `ready`'s tight bounds.
- **`set_active(false)` also visually switches to `S_inactive`**, and
  `set_active(true)` back to `S_ready` — this is `PGButton`'s override of the
  base `PGItem::set_active()`, which by itself has no visual effect.

## Notify interface — `PGButtonNotify`

Extends [PGItemNotify](PGItem.md#notify-interface--pgitemnotify) with one more
callback:

```cpp
class PGButtonNotify : public PGItemNotify {
protected:
  virtual void button_click(PGButton *button, const MouseWatcherParameter &param);
};
```

Attach with `button->set_notify(PGButtonNotify*)` (shadows/narrows
`PGItem::set_notify`, so `get_notify()` returns `PGButtonNotify*` directly).
`PGSliderBar` uses this internally to watch its own thumb/left/right buttons.

## API

### State enum
```cpp
enum State { S_ready = 0, S_depressed, S_rollover, S_inactive };
```

### Setup
| Signature | Notes |
|---|---|
| `void setup(const std::string &label, PN_stdfloat bevel = 0.1f)` | Default text-label button; see "Behavior notes" |
| `void setup(const NodePath &ready)` / `setup(ready, depressed)` / `setup(ready, depressed, rollover)` | Convenience overloads; missing states default to `ready` |
| `void setup(const NodePath &ready, depressed, rollover, inactive)` | Full custom-geometry setup |

### Click buttons
| Signature | Notes |
|---|---|
| `bool add_click_button(const ButtonHandle&)` | Returns false if already present |
| `bool remove_click_button(const ButtonHandle&)` | Returns false if not present |
| `bool has_click_button(const ButtonHandle&)` | |
| `bool is_button_down()` | Whether the button is currently held depressed |

### Events / notify
| Signature | Notes |
|---|---|
| `static std::string get_click_prefix()` | `"click-"` |
| `std::string get_click_event(const ButtonHandle&) const` | `"click-" + button.get_name() + "-" + get_id()` |
| `void set_notify(PGButtonNotify*)` / `PGButtonNotify *get_notify() const` | |

Also overrides `set_active(bool)` from `PGItem` (see Behavior notes).

## Events

| Event | Fires when |
|---|---|
| `click-<button>-<id>` | Button pressed and released normally over the button (see rules above) |

Plus all base [PGItem events](PGItem.md#events) (`enter-`, `exit-`, `press-`,
`release-`, etc.) — `click-` is additional, not a replacement.

## Usage

```cpp
PT(PGButton) btn = new PGButton("ok");
btn->setup("OK");
top_np.attach_new_node(btn);

// React via event:
accept(btn->get_click_event(MouseButton::one()), [](const Event *) { ... });

// Or via Notify:
class MyPanel : public PGButtonNotify {
  void button_click(PGButton *b, const MouseWatcherParameter &p) override { ... }
};
```

## See also

[PGItem.md](PGItem.md) (base class, frame/state concepts) ·
[PGSliderBar.md](PGSliderBar.md) (composes 3 `PGButton`s) · [README.md](README.md)
