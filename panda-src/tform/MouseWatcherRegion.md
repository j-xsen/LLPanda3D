# MouseWatcherRegion

**Source:** `panda/src/tform/mouseWatcherRegion.h` / `.I` / `.cxx`
**Inherits:** `TypedWritableReferenceCount`, `Namable`
**Held by:** [MouseWatcherBase](MouseWatcherBase.md) (and therefore
[MouseWatcherGroup](MouseWatcherGroup.md) / [MouseWatcher](MouseWatcher.md))

`MouseWatcherRegion` defines one named, rectangular, axis-aligned region in
normalized screen space (`left, right, bottom, top`, same convention as
`DisplayRegion`: 0,0 lower-left, arbitrary scale). A
[MouseWatcher](MouseWatcher.md) tests the live mouse position against every
region it holds and dispatches the virtual callback hooks below — subclass
and override the ones you care about (all are empty no-ops by default).

## Behavior notes

- **`operator<` defines "preferred region" ordering, not spatial order.**
  Higher `_sort` wins; among equal `_sort`, the *smaller* `_area` wins. This
  is what [MouseWatcher](MouseWatcher.md)'s `get_preferred_region()` uses to
  pick a single "entered" region out of several overlapping ones — a small
  region nested inside a big one is preferred automatically without needing
  an explicit sort value, as long as both have the same sort.
- **`_area` is a cached field, not computed on demand.** `set_frame()`
  recomputes `_area = (right-left)*(top-bottom)` every time the frame is
  set — it is not recomputed lazily, so directly assigning `_frame` (there's
  no way to via the public API, but relevant if subclassing) would leave
  `_area` stale and silently corrupt the sort-preference logic above.
- **`set_keyboard(true)` bypasses the mouse-position check entirely** —
  such a region receives `press()`/`release()`/`keystroke()` for *every*
  keyboard event regardless of where the mouse is, alongside whatever region
  the mouse happens to be over. It's the mechanism used for global
  hotkeys/focus-independent keyboard listeners.
- **`SuppressFlags` are checked by the mouse watcher, not by the region
  itself** — `set_suppress_flags(SF_mouse_position)` on a region that
  currently owns the mouse tells [MouseWatcher](MouseWatcher.md) to stop
  passing `xy`/`pixel_xy` down the data graph while the mouse is over this
  region (see [MouseWatcher](MouseWatcher.md) behavior notes); the region
  class only stores the flags, it does no suppression itself.
- **The nine callback hooks are all empty `virtual` no-ops in the base
  class** (`enter_region`, `exit_region`, `within_region`, `without_region`,
  `press`, `release`, `keystroke`, `candidate`, `move`) — a subclass
  overrides only the ones it needs; there's no need to call the base
  implementation first.
- **`enter_region`/`exit_region` fire only for the single "preferred"
  region**, while `within_region`/`without_region` fire for *every*
  region the mouse geometrically overlaps, including nested ones — see
  [MouseWatcher](MouseWatcher.md)'s `set_current_regions()` for exactly how
  the two sets diverge.

## API

### Construction (PUBLISHED, inline)
| Signature | Notes |
|---|---|
| `MouseWatcherRegion(const std::string &name, PN_stdfloat left, right, bottom, top)` | |
| `MouseWatcherRegion(const std::string &name, const LVecBase4 &frame)` | `frame` is `(left, right, bottom, top)` |

### Frame / sort / flags (PUBLISHED, inline)
| Signature | Notes |
|---|---|
| `void set_frame(...)` / `const LVecBase4 &get_frame() const` | Region bounds; also recomputes `get_area()` |
| `PN_stdfloat get_area() const` | Cached, see behavior notes |
| `void set_sort(int)` / `int get_sort() const` | Overlap-resolution priority; higher wins, default 0 |
| `void set_active(bool)` / `bool get_active() const` | Inactive regions are never "over"; may still get keyboard events if `get_keyboard()` |
| `void set_keyboard(bool)` / `bool get_keyboard() const` | Opt in to global keyboard event delivery |
| `void set_suppress_flags(int)` / `int get_suppress_flags() const` | Bitmask of `SF_mouse_button`\|`SF_other_button`\|`SF_mouse_position` |

### Callback hooks (public, virtual)
| Signature | Notes |
|---|---|
| `virtual void enter_region(const MouseWatcherParameter &)` | Mouse becomes "entered" (preferred) in this region |
| `virtual void exit_region(const MouseWatcherParameter &)` | Mouse is no longer the preferred region |
| `virtual void within_region(const MouseWatcherParameter &)` | Mouse moves within bounds (fires for nested regions too) |
| `virtual void without_region(const MouseWatcherParameter &)` | Mouse moves fully outside bounds |
| `virtual void press(const MouseWatcherParameter &)` / `release(...)` | Mouse or keyboard button down/up while region is relevant |
| `virtual void keystroke(const MouseWatcherParameter &)` | A semantic keystroke (see [MouseWatcherParameter](MouseWatcherParameter.md)) |
| `virtual void candidate(const MouseWatcherParameter &)` | IME candidate string update |
| `virtual void move(const MouseWatcherParameter &)` | Mouse moved while a button is down over this region |

## Usage

```cpp
class ClickRegion : public MouseWatcherRegion {
public:
  ClickRegion(const std::string &name, const LVecBase4 &frame)
    : MouseWatcherRegion(name, frame) {}

  void press(const MouseWatcherParameter &param) override {
    if (param.has_button() && param.get_button() == MouseButton::one()) {
      nout << get_name() << " clicked\n";
    }
  }
};

PT(ClickRegion) region = new ClickRegion("button1", LVecBase4(-0.5, 0.5, -0.2, 0.2));
mouse_watcher->add_region(region);  // mouse_watcher: PT(MouseWatcher)
```

## See also

[MouseWatcherParameter](MouseWatcherParameter.md) (callback argument) ·
[MouseWatcherBase](MouseWatcherBase.md) (holds a sorted collection of
these) · [MouseWatcher](MouseWatcher.md) (drives the callbacks)
