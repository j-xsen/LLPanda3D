# PGWaitBar

**Source:** `panda/src/pgui/pgWaitBar.h` / `.I` / `.cxx`
**Inherits:** [PGItem](PGItem.md) → `PandaNode`

A simple non-interactive progress/loading bar: a background frame with a
second, separately-styled bar drawn on top that grows from left to right as
`value` approaches `range`.

## Behavior notes

- **Only one state (state `0`) is used.** `setup()` calls `set_state(0)` and
  builds a single frame style; there's no ready/active/etc. distinction like
  the interactive widgets.
- **The fill bar is regenerated lazily**, only when `_bar_state != get_state()`
  — since state never changes after `setup()`, in practice this means the bar
  regenerates once after any `set_value()`/`set_range()`/`set_bar_style()`
  call (each of those sets `_bar_state = -1` to force it) and is cached
  otherwise. This is driven from `cull_callback()`, i.e. it updates once per
  frame at most, not synchronously on the setter call.
- **The fill bar is inset by the *background* frame style's border width**,
  not its own — `update()` reads `get_frame_style(state).get_width()` to
  compute the inset before drawing the bar via `_bar_style.generate_into(...)`.
  Setting a `bar_style` with a nonzero `width` of its own has no effect on this
  inset.
- **`value` is not clamped by the setter** — `set_value()` stores whatever you
  pass; only the *rendered fraction* is clamped to `[0, 1]` in `update()`.
  `get_value()`/`get_percent()` will report out-of-range numbers if you set
  them that way.
- **If `value == 0` or `range == 0`, no bar geometry is generated at all**
  (not even a zero-width one).

## API

### Setup
| Signature | Notes |
|---|---|
| `void setup(PN_stdfloat width, PN_stdfloat height, PN_stdfloat range)` | Builds background (bevel-in) + bar (bevel-out) frame styles, centers frame at origin |

### Value
| Signature | Notes |
|---|---|
| `void set_range(PN_stdfloat)` / `PN_stdfloat get_range() const` | The value at which the bar reads 100% |
| `void set_value(PN_stdfloat)` / `PN_stdfloat get_value() const` | Not clamped; should be `[0, range]` |
| `PN_stdfloat get_percent() const` | `(value / range) * 100` |
| `void set_bar_style(const PGFrameStyle&)` / `PGFrameStyle get_bar_style() const` | Style of the foreground fill bar (background style is the normal per-state `PGItem` frame style) |

## Usage

```cpp
PT(PGWaitBar) bar = new PGWaitBar("loading");
bar->setup(4.0f, 0.5f, 100.0f);   // width, height, range
top_np.attach_new_node(bar);

bar->set_value(37.0f);   // ~37% filled next frame
```

## See also

[PGItem.md](PGItem.md) · [PGFrameStyle.md](PGFrameStyle.md) · [README.md](README.md)
