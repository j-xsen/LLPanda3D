# ScissorAttrib

**Source:** `panda/src/pgraph/scissorAttrib.h` (+ `.I`, `.cxx`)
**Inherits:** RenderAttrib

Restricts rendering to a rectangular region within the enclosing
`DisplayRegion`, in screen space — akin to OpenGL's `glScissor()`. Geometry
outside the region isn't rendered, but it also isn't *culled* (the cull
traversal still visits it; the GSG clips pixels at rasterization
time instead), and the viewport/lens are unaffected.

**Distinct from [ScissorEffect](ScissorEffect.md):** `ScissorEffect`
defines its region relative to 2-D/3-D scene-graph coordinates and *does*
participate in culling; `ScissorAttrib` is a flat screen-space rectangle
with no culling interaction. `ScissorAttrib` suits a fixed on-screen
clip region (e.g. split-screen); `ScissorEffect` suits a region tied to
something in the scene.

## Behavior notes

- Frame coordinates are clamped into `[0,1]` and `right`/`top` are clamped
  to be `>= left`/`>= bottom` respectively, in the constructor — an
  inverted or out-of-range frame can't be constructed.
- `compose_impl()`: composing two on `ScissorAttrib`s **intersects** their
  frames (`max` of left/bottom, `min` of right/top) rather than the usual
  "downstream wins" — one of the few attribs (with `RenderModeAttrib`'s
  `M_unchanged`) that doesn't replace outright on compose. An off attrib
  composed with anything yields the other attrib unchanged.
- `make_off()`/`make_default()` share one memoized static instance
  (`_off_attrib`); `make_default()` is literally `return make_off();`.
- Bam I/O: the `_off` flag is only written/read for files at minor version
  ≥ 34 — pre-34 files have no off state (always treated as on).

## API

| Signature | Notes |
|---|---|
| `static CPT(RenderAttrib) make(PN_stdfloat left, right, bottom, top)` | Frame in `[0,1]` DisplayRegion-relative coords |
| `static CPT(RenderAttrib) make(const LVecBase4 &frame)` | Same, packed as `(left, right, bottom, top)` |
| `static CPT(RenderAttrib) make_off()` | Disables scissoring (full DisplayRegion) |
| `static CPT(RenderAttrib) make_default()` | Same as `make_off()` |
| `bool is_off() const` | |
| `const LVecBase4 &get_frame() const` | |

## Usage

```cpp
node_path.set_attrib(ScissorAttrib::make(0.0f, 0.5f, 0.0f, 1.0f)); // left half of DisplayRegion
```

## See also

[README — the state pipeline](README.md#the-state-pipeline),
[RenderAttrib](RenderAttrib.md), [ScissorEffect](ScissorEffect.md)
