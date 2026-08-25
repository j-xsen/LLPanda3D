# MouseInterfaceNode

**Source:** `panda/src/tform/mouseInterfaceNode.h` / `.I` / `.cxx`
**Inherits:** `DataNode`
**Inherited by:** [DriveInterface](DriveInterface.md), [Trackball](Trackball.md), [MouseSubregion](MouseSubregion.md)

`MouseInterfaceNode` is the common base for [data graph](../dgraph/README.md)
nodes that watch the mouse and keyboard and act only when some combination of
buttons is (or isn't) held down. It contributes a single `button_events`
input wire and a small "required buttons" gate that subclasses consult before
doing their real work in `do_transmit_data()`.

## Behavior notes

- **The button gate is a mask + expected-state pair, not a callback list.**
  `require_button(button, is_down)` adds the button to
  `_required_buttons_mask` and records the wanted state in
  `_required_buttons_state`. `check_button_events()` computes
  `required_buttons_match` by masking the live `_current_button_state`
  against that mask and comparing to the expected state — every required
  button must match simultaneously, there is no OR-of-requirements mode.
- **`watch_button()` and `require_button()` share the same underlying mask.**
  Calling `watch_button(b)` also adds `b` to `_required_buttons_mask`, but
  leaves it "up" in `_required_buttons_state` — i.e. it does not, by itself,
  make the button *required*; it only makes `is_down(b)` queryable via
  `_current_button_state`. A subsequent `require_button(b, true)` on a
  watched button changes only the expected state, not the mask membership.
- **`clear_button()` only fully removes a button if it isn't also watched.**
  It clears the button from both the mask and the expected state, but if the
  button is present in `_watched_buttons`, the mask/state are rebuilt to
  keep it (masked "up") rather than dropping it — so a `watch_button()`ed
  button can never be fully unmasked by `clear_button()`, only by never
  having required it in the first place.
- **`check_button_events()` must be called every frame a subclass wants
  accurate button state**, even if it discards the return value — it's the
  only place `_current_button_state` is updated (via
  `ButtonEventList::update_mods()`), and it's also what recomputes
  `required_buttons_match`. Skipping the call in some frames means both go
  stale.
- **Returns `nullptr` for `button_events` when no events arrived this
  frame** (`input.has_data()` false on the `button_events` wire), which is
  the normal case whenever nothing was pressed/released — callers must
  handle the null case rather than assuming an always-populated list.

## API

### Construction
| Signature | Notes |
|---|---|
| `explicit MouseInterfaceNode(const std::string &name)` | Defines the `button_events` input wire |

### Button requirements (PUBLISHED)
| Signature | Notes |
|---|---|
| `void require_button(const ButtonHandle &button, bool is_down)` | Adds/updates a required button+state; `do_transmit_data()` in subclasses should no-op unless all requirements hold |
| `void clear_button(const ButtonHandle &button)` | Removes one requirement (unless the button is also `watch_button()`ed) |
| `void clear_all_buttons()` | Removes all requirements set via `require_button()`, but preserves watched buttons |

### For subclasses (protected)
| Signature | Notes |
|---|---|
| `void watch_button(const ButtonHandle &button)` | Makes `is_down()` track this button, without requiring it to be down |
| `const ButtonEventList *check_button_events(const DataNodeTransmit &input, bool &required_buttons_match)` | Call once per `do_transmit_data()`; updates button state and returns this frame's events (or `nullptr`) |
| `INLINE bool is_down(ButtonHandle button) const` | Current state of a `watch_button()`ed (or required) button |

## Usage

```cpp
class MyDevice : public MouseInterfaceNode {
public:
  explicit MyDevice(const std::string &name) : MouseInterfaceNode(name) {
    watch_button(MouseButton::one());
    // Only do anything while the control key is held.
    require_button(KeyboardButton::control(), true);
  }

protected:
  void do_transmit_data(DataGraphTraverser *, const DataNodeTransmit &input,
                         DataNodeTransmit &output) override {
    bool required_buttons_match;
    check_button_events(input, required_buttons_match);
    if (!required_buttons_match) {
      return;
    }
    if (is_down(MouseButton::one())) {
      // ... act on the drag ...
    }
  }
};
```

## See also

[DataNode](../dgraph/DataNode.md) (base class, defines the input/output
wire protocol) · [DriveInterface](DriveInterface.md),
[Trackball](Trackball.md), [MouseSubregion](MouseSubregion.md) (concrete
subclasses)
