# MouseSubregion

**Source:** `panda/src/tform/mouseSubregion.h` / `.I` / `.cxx`
**Inherits:** [MouseInterfaceNode](MouseInterfaceNode.md)

`MouseSubregion` rescales the `xy`/`pixel_xy`/`pixel_size` [data
graph](../dgraph/README.md) wires from a rectangular sub-area of the window
into full-window-equivalent coordinates, so that a
[MouseWatcher](MouseWatcher.md) (or anything else expecting normalized
`(-1..1)` mouse coordinates) parented below it perceives that sub-area as if
it were the entire window. Choosing dimensions that match a `DisplayRegion`
gives that `DisplayRegion` its own independent virtual mouse.

## Behavior notes

- **Coordinates outside the sub-rectangle are dropped, not clamped.**
  `do_transmit_data()` rescales the incoming `xy` into `n`, then only
  forwards `xy`/`pixel_xy` downstream if `n` falls within `[-1, 1]` on both
  axes; otherwise `has_mouse` stays false for that frame and neither wire is
  set at all — a [MouseWatcher](MouseWatcher.md) below it correctly sees "no
  mouse" rather than a mouse clamped to its edge.
- **Button events are gated on the same in/out-of-bounds test, with an
  asymmetric rule for "up" events.** When the mouse is within bounds, *all*
  button events pass through unchanged. When it's outside, only `T_up`
  events are forwarded (and only those — down/repeat/keystroke events are
  dropped) — this prevents a "stuck button" downstream: if the mouse drags
  outside the subregion while a button is held, the eventual release still
  reaches whatever grabbed the earlier press.
- **`pixel_size` is proportionally scaled to the subregion's own fraction
  of the window**, not read from a fixed value: `n = s * (_r - _l), s *
  (_t - _b)`, where `s` is the real window's `pixel_size`. `set_dimensions()`
  takes fractions in `[0, 1]` (matching `DisplayRegion`'s convention, not
  normalized `-1..1`), so a subregion covering the left half of the window
  is `set_dimensions(0, 0.5, 0, 1)`.
- **The output `pixel_xy` is only recomputed when a `pixel_size` input is
  actually available that frame** — if `pixel_size` isn't present on the
  input side (unusual, but possible depending on what's parented above),
  the rescaled `xy` is still forwarded but `pixel_xy` silently is not.

## API

| Signature | Notes |
|---|---|
| `explicit MouseSubregion(const std::string &name)` | Defines `pixel_xy`/`pixel_size`/`xy`/`button_events` in/out wire pairs |
| `void set_dimensions(PN_stdfloat l, r, b, t)` | Fractions `0..1` of the window, `DisplayRegion` convention |
| `PN_stdfloat get_left/right/bottom/top() const` | Read back the last-set dimensions |

## Usage

```cpp
NodePath mak_np = data_root.attach_new_node(new MouseAndKeyboard(win, 0, "mak"));

PT(MouseSubregion) sub = new MouseSubregion("left_half");
sub->set_dimensions(0.0, 0.5, 0.0, 1.0);
NodePath sub_np = mak_np.attach_new_node(sub);

// A MouseWatcher parented here only sees mouse activity in the left half
// of the window, remapped to the full -1..1 range.
NodePath watcher_np = sub_np.attach_new_node(new MouseWatcher("left_watcher"));
```

## See also

[MouseInterfaceNode](MouseInterfaceNode.md) (base class) ·
[MouseWatcher](MouseWatcher.md) (typical downstream consumer) ·
[display/MouseAndKeyboard](../display/MouseAndKeyboard.md) (typical
upstream source of the wires this rescales)
