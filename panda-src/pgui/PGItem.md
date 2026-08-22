# PGItem

**Source:** `panda/src/pgui/pgItem.h` / `.I` / `.cxx` (+ `pgItemNotify.h/.I/.cxx`)
**Inherits:** `PandaNode`
**Inherited by:** [PGButton](PGButton.md), [PGEntry](PGEntry.md), [PGSliderBar](PGSliderBar.md), [PGWaitBar](PGWaitBar.md), [PGVirtualFrame](PGVirtualFrame.md) (→ [PGScrollFrame](PGScrollFrame.md))

`PGItem` is the abstract base for every PGUI widget. It is a `PandaNode`
representing a rectangular clickable region ("frame") plus a set of alternate
render subgraphs ("state defs"), and it owns a [PGMouseWatcherRegion](PGMouseWatcherRegion.md)
that registers it with the `MouseWatcher` input system whenever it's parented
under a [PGTop](PGTop.md). See [README.md](README.md) for the shared concepts
(frame vs. state defs, event naming, notify interfaces) — this doc covers
`PGItem`'s specific members.

## Behavior notes

- **A `PGItem` with no frame is invisible to the mouse but still renders.**
  `has_frame()` false means no region is ever activated, so enter/exit/press
  events never fire, even if the node has visible geometry.
- **`get_id()` drives every event name and defaults to the region's
  auto-generated name** (`"pg" + sequence_number`), not the node's `get_name()`.
  Call `set_id()` if you need predictable/human-readable event names; the
  region name and thus every event name changes immediately.
- **Only one `PGItem` in the whole application can have keyboard focus at a
  time** (`_focus_item` is a static). Calling `set_focus(true)` on one item
  automatically calls `set_focus(false)` on whichever item had it.
  `set_active(false)` implicitly clears focus too.
- **`background_focus` is different from `focus`.** An item with
  `set_background_focus(true)` receives `press()`/`release()`/`keystroke()`/
  `candidate()` calls (with `background=true`) for *every* keypress in the
  application, regardless of normal focus or even visibility — driven by
  [PGMouseWatcherBackground](PGMouseWatcherBackground.md). Many items may have
  background focus simultaneously; this is how e.g. a chat-open hotkey can
  work even while some other widget has normal focus.
  When `background=true`, base `PGItem` methods do **not** throw the normal
  press/release/keystroke events or play sounds — only the Notify callback
  fires (see `press()`/`release()`/`keystroke()` below). `candidate()` never
  throws a sound event for either case.
- **State defs are lazily created and pipeline-cycle aware.** `get_state_def(n)`
  creates a `NodePath` for state `n` the first time it's called; the frame
  style geometry for that state is (re)generated on demand and cached, marked
  stale by `frame_changed()` / `set_frame_style()`.
- **`instance_to_state_def()`** is the normal way to put your own geometry into
  a state — it instances your `NodePath` under the (possibly not-yet-existing)
  state-def root, rather than you calling `get_state_def()` and reparenting
  manually.
- **The `press`/`repeat` distinction:** `press()` throws a `repeat-` event
  instead of `press-` when `param.is_keyrepeat()` is true (OS key-repeat), but
  the Notify callback (`item_press`) fires the same way either way.
- **Copying a `PGItem`** (`make_copy()` / `NodePath::copy_to()`) deep-copies all
  state defs but does *not* copy the notify pointer, and gives the copy's
  internal region the *same* name as the original's — so a copied item
  generates identical event names to the original.

## Notify interface — `PGItemNotify`

`PGItemNotify` (`pgItemNotify.h`) is a protected-virtual callback mixin. Any
C++ object that derives from it can be attached with `item->set_notify(this)`
to receive direct calls instead of/in addition to string events; on
destruction of either side, the link is automatically severed
(`PGItemNotify::~PGItemNotify()` detaches all watched items; `PGItem::~PGItem()`
detaches its own notify pointer). Not owning — no reference counting on either
side.

```cpp
class MyWatcher : public PGItemNotify {
protected:
  void item_press(PGItem *item, const MouseWatcherParameter &param) override {
    // react directly, no event-string round trip
  }
};
```

Overridable methods (all no-ops by default): `item_transform_changed`,
`item_frame_changed`, `item_draw_mask_changed`, `item_enter`, `item_exit`,
`item_within`, `item_without`, `item_focus_in`, `item_focus_out`, `item_press`,
`item_release`, `item_keystroke`, `item_candidate`, `item_move` — each mirrors
the correspondingly-named `PGItem` hook and receives the watched `PGItem*` as
first argument.

## API

### Construction
| Signature | Notes |
|---|---|
| `explicit PGItem(const std::string &name)` | `name` is the node name; the region gets its own auto-generated id (see `set_id()`) |

### Frame
| Signature | Notes |
|---|---|
| `void set_frame(PN_stdfloat left, right, bottom, top)` / `set_frame(const LVecBase4&)` | Sets the clickable rectangle, local coords |
| `const LVecBase4 &get_frame() const` | Asserts `has_frame()` |
| `bool has_frame() const` | |
| `void clear_frame()` | Removes the frame; item becomes unclickable |
| `LMatrix4 get_frame_inv_xform() const` | Inverse of the last-computed screen transform; used to map mouse coords back into frame space |

### State
| Signature | Notes |
|---|---|
| `void set_state(int state)` / `int get_state() const` | Selects which state def is rendered |
| `int get_num_state_defs() const` | One more than the highest state index ever touched (may have holes) |
| `bool has_state_def(int state) const` | |
| `void clear_state_def(int state)` | Resets a state's subgraph to empty |
| `NodePath &get_state_def(int state)` | Lazily creates/returns the render root for a state |
| `NodePath instance_to_state_def(int state, const NodePath &path)` | Instances `path` under the state's root — the normal way to add custom geometry |
| `PGFrameStyle get_frame_style(int state)` / `void set_frame_style(int state, const PGFrameStyle &style)` | Border/background style for that state — see [PGFrameStyle](PGFrameStyle.md) |

### Active / focus
| Signature | Notes |
|---|---|
| `virtual void set_active(bool)` / `bool get_active() const` | Whether the item responds to mouse events at all (independent of visual "inactive" state, which subclasses manage themselves) |
| `virtual void set_focus(bool)` / `bool get_focus() const` | Keyboard focus; only one item app-wide at a time; no-op if `!get_active()` |
| `void set_background_focus(bool)` / `bool get_background_focus() const` | See "Behavior notes" above |
| `void set_suppress_flags(int)` / `int get_suppress_flags() const` | Passthrough to the underlying `MouseWatcherRegion::set_suppress_flags()` — blocks lower-priority regions/camera-control from also seeing the event |
| `static PGItem *get_focus_item()` | The one item with focus, or `nullptr` |

### Identity / events
| Signature | Notes |
|---|---|
| `const std::string &get_id() const` / `void set_id(const std::string&)` | Backing string for every event name; defaults to an auto-generated region name |
| `static std::string get_enter_prefix()` etc. | Prefix constants: `enter-`, `exit-`, `within-`, `without-`, `fin-` (focus_in), `fout-` (focus_out), `press-`, `repeat-`, `release-`, `keystroke-` |
| `std::string get_enter_event() const` etc. | `prefix + get_id()` (or `prefix + button_name + "-" + get_id()` for press/repeat/release) |

### Sounds (only if `HAVE_AUDIO`)
| Signature | Notes |
|---|---|
| `void set_sound(const std::string &event, AudioSound *sound)` | Plays `sound` whenever `event` fires via this item's own throw calls (enter/exit/within/without/focus_in/focus_out/press/repeat/release/keystroke) |
| `void clear_sound(const std::string &event)` / `AudioSound *get_sound(...)` / `bool has_sound(...)` | |

### Text node default
| Signature | Notes |
|---|---|
| `static TextNode *get_text_node()` / `static void set_text_node(TextNode*)` | The shared `TextNode` used to generate default button/entry labels; lazily created (black, left-aligned) if never set |

### Notify
| Signature | Notes |
|---|---|
| `void set_notify(PGItemNotify *notify)` / `bool has_notify() const` / `PGItemNotify *get_notify() const` | See "Notify interface" above |

## Virtual event hooks (override in subclasses; base impl throws the corresponding event + calls notify)

`enter_region`, `exit_region`, `within_region`, `without_region`, `focus_in`,
`focus_out`, `press(param, background)`, `release(param, background)`,
`keystroke(param, background)`, `candidate(param, background)`, `move(param)`.
Also four `static` `background_*` dispatchers that fan a background event out
to every item currently in `set_background_focus(true)` that doesn't itself
have normal focus (used by [PGMouseWatcherBackground](PGMouseWatcherBackground.md)).

## Events

| Event | Fires when | Params (`EventParameter`s) |
|---|---|---|
| `enter-<id>` | mouse enters the frame (not a nested frame) | `PGMouseWatcherParameter` |
| `exit-<id>` | mouse exits the frame, or enters a nested frame | `PGMouseWatcherParameter` |
| `within-<id>` | mouse moves within the frame, incl. nested frames | `PGMouseWatcherParameter` |
| `without-<id>` | mouse moves completely outside the frame | `PGMouseWatcherParameter` |
| `fin-<id>` | item gains keyboard focus | *(none)* |
| `fout-<id>` | item loses keyboard focus | *(none)* |
| `press-<button>-<id>` | button pressed while mouse is within the frame (foreground only) | `PGMouseWatcherParameter` |
| `repeat-<button>-<id>` | same, but OS key-repeat rather than a fresh press | `PGMouseWatcherParameter` |
| `release-<button>-<id>` | button released (foreground only) | `PGMouseWatcherParameter` |
| `keystroke-<id>` | any key typed while focused (foreground only) | `PGMouseWatcherParameter` |

All PG events carry a single `PGMouseWatcherParameter` (a `TypedWritableReferenceCount`-wrapped `MouseWatcherParameter` — see [PGMouseWatcherParameter.md](PGMouseWatcherParameter.md)) except `fin-`/`fout-`, which carry nothing.

## Usage

```cpp
// A PGItem is abstract in practice — use a concrete subclass. Shown here:
// generic frame/event wiring shared by all of them.
PT(PGButton) btn = new PGButton("ok");
btn->setup("OK");                       // concrete-class specific
btn->set_id("ok_button");               // optional: readable event names
top_np.attach_new_node(btn);            // top_np must be under a PGTop, see PGTop.md

// Preferred for application code: listen for the event (btn->get_click_event(...)).

// Alternative for C++-internal wiring: Notify interface, no event strings.
class MyPanel : public PGItemNotify {
  void item_press(PGItem *item, const MouseWatcherParameter &param) override { ... }
};
```

## See also

[README.md](README.md) for shared PGUI concepts · [PGTop.md](PGTop.md) (required
parent) · [PGFrameStyle.md](PGFrameStyle.md) (frame style value type) ·
[PGMouseWatcherRegion.md](PGMouseWatcherRegion.md) (the region this class owns) ·
[PGMouseWatcherParameter.md](PGMouseWatcherParameter.md) (event payload)
