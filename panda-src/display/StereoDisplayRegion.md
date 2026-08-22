# StereoDisplayRegion

**Source:** `panda/src/display/stereoDisplayRegion.h` (+ `.I`, `.cxx`)
**Inherits:** [DisplayRegion](DisplayRegion.md) **Inherited by:** (none)

A `DisplayRegion` that wraps a left-eye/right-eye pair of ordinary
`DisplayRegion`s and forwards nearly every operation to both. It has no
physical rectangle of its own within the window — it *pretends* to (so code
that just wants "a `DisplayRegion`" can treat it like one) but every real
setting lives on the two child regions it owns. Constructor is `protected`
— created only via `GraphicsOutput::make_stereo_display_region()`, and
whether a window/buffer with the stereo property gets one of these by
default is controlled by the `default-stereo-camera` config variable
(documented in the module [README.md](README.md)).

## Behavior notes

- **Most setters just fan out unchanged to both eyes** — `set_camera()`,
  `set_incomplete_render()`, `set_texture_reload_priority()`,
  `set_cull_traverser()`, `set_target_tex_page()`, `set_clear_value()`,
  `disable_clears()`, `set_pixel_zoom()`, `set_dimensions()` — call the base
  `DisplayRegion` implementation on `this` (so queries against the stereo
  object itself stay consistent) and then repeat the call on `_left_eye`
  and `_right_eye`.
- **`set_clear_active()` treats the color buffer specially.** The clear-active
  flag for `RTP_color` is set only on the parent stereo region (not
  propagated to the right eye); every *other* buffer type (depth, stencil,
  etc.) is additionally set on `_right_eye`, "on the assumption that we want
  to clear these buffers between drawing the eyes, and that the right eye
  is the second of the pair" (from the source comment) — clearing depth
  once per eye pair, not once per eye, would let the second eye's geometry
  incorrectly depth-test against the first eye's depth buffer.
- **`set_sort(n)` offsets the two eyes** — `n` on the stereo region itself,
  `n+1` on the left eye, `n+2` on the right eye — guaranteeing the left eye
  draws before the right eye regardless of what sort value the caller
  chose, without the caller needing to think about it.
- **`set_stereo_channel()` is the real per-eye activation switch**, not just
  a passive flag — see the four-way table below. Calling it while the
  stereo region is inactive (`!is_active()`) is a no-op (early return)
  beyond recording the channel on the base class; `set_active(true)`
  re-applies `set_stereo_channel(get_stereo_channel())` to re-derive the
  correct per-eye active state.

  | `stereo_channel` | left eye | right eye |
  |---|---|---|
  | `SC_stereo` | `SC_left`, active | `SC_right`, active |
  | `SC_left` | `SC_left`, active | inactive |
  | `SC_right` | inactive | `SC_right`, active |
  | `SC_mono` | `SC_mono`, active | inactive |

- **`set_tex_view_offset(n)` sets `n` on the left eye and `n+1` on the right
  eye** — the standard convention for stereo textures (view 0 = left, view
  1 = right), building on the same convention `DisplayRegion::set_stereo_channel()`
  already applies automatically for a plain `DisplayRegion`.
- **The `nassertv` in the constructor requires both child regions to
  already belong to the same window** as the `StereoDisplayRegion` itself —
  it doesn't create the left/right regions, just wraps two that
  `GraphicsOutput` already made.
- **`make_cull_result_graph()` composes, rather than delegates to, the base
  implementation** — it builds a `"stereo"` root `PandaNode` with two named
  children, `"left"` and `"right"`, each populated by calling
  `make_cull_result_graph()` on the corresponding eye and attached at that
  eye's own `sort` value.

## API

All of these override the corresponding `DisplayRegion`/`DrawableRegion`
virtual and fan out to both eyes as described above, except where noted:

| Signature | Notes |
|---|---|
| `virtual void set_clear_active(int n, bool clear_active)` | Special-cases `RTP_color` — see behavior notes. |
| `virtual void set_clear_value(int n, const LColor&)` | |
| `virtual void disable_clears()` | |
| `virtual void set_pixel_zoom(PN_stdfloat)` | |
| `virtual void set_dimensions(int i, const LVecBase4&)` | |
| `virtual bool is_stereo() const` | Returns `true` (vs. `false` on plain `DisplayRegion`). |
| `virtual void set_camera(const NodePath&)` | |
| `virtual void set_active(bool)` | Re-derives per-eye active state via `set_stereo_channel()` when reactivated. |
| `virtual void set_sort(int)` | Offsets left (+1) and right (+2). |
| `virtual void set_stereo_channel(Lens::StereoChannel)` | The real per-eye switch — see table above. |
| `virtual void set_tex_view_offset(int)` | Left = n, right = n+1. |
| `virtual void set_incomplete_render(bool)` | |
| `virtual void set_texture_reload_priority(int)` | |
| `virtual void set_cull_traverser(CullTraverser*)` | |
| `virtual void set_target_tex_page(int)` | |
| `virtual void output(std::ostream&) const` | Prints via the left eye's own `output()`. |
| `virtual PT(PandaNode) make_cull_result_graph()` | Composes a `"stereo"` root over both eyes' graphs — see behavior notes. |
| `INLINE DisplayRegion *get_left_eye()` / `INLINE DisplayRegion *get_right_eye()` | Direct access to the underlying per-eye `DisplayRegion`s for anything not covered by the fan-out API above. |

## Usage

```cpp
// A window/buffer created with the stereo framebuffer property, with
// default-stereo-camera true, already has a StereoDisplayRegion as its
// default 3-d region. To access the eyes independently:
StereoDisplayRegion *sdr = DCAST(StereoDisplayRegion, window->get_display_region_3d());
sdr->set_camera(stereo_camera_np);
sdr->get_left_eye()->set_clear_color(LColor(0, 0, 0, 1));
```

## See also

- [DisplayRegion.md](DisplayRegion.md) — base class; most of this class's behavior is "do what DisplayRegion does, twice."
- [GraphicsOutput.md](GraphicsOutput.md) — `make_stereo_display_region()`.
- [README.md](README.md) — `default-stereo-camera`, `side-by-side-stereo`, `red-blue-stereo`, `swap-eyes` config variables.
