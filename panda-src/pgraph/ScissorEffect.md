# ScissorEffect

**Source:** `panda/src/pgraph/scissorEffect.h` (+ `.I`, `.cxx`)
**Inherits:** RenderEffect

A higher-level wrapper around [ScissorAttrib](ScissorAttrib.md) that lets
you define the scissor region as world-space points relative to a node
(projected to screen space at cull time), instead of `ScissorAttrib`'s raw
screen-fraction rectangle — and additionally performs view-frustum culling
against that region, not just GPU-level scissoring.

## Distinction from ScissorAttrib

`ScissorAttrib` is the low-level `RenderAttrib` that directly holds a
`left, right, bottom, top` fractional screen rectangle and composes down
the state stack the normal attrib way (each nested `ScissorAttrib`
intersects with its parent's region — see `ScissorAttrib.md`).
`ScissorEffect` is node-local, cull-time behavior: it computes a screen
rectangle by projecting a set of node-relative 3D points every frame (as
the node/camera move, the rectangle is recomputed), *and* additionally
narrows the cull traverser's view frustum to that region so off-screen
geometry under the node is culled early — not just clipped at the pixel
level. When `get_clip()` is true, `cull_callback()` also emits an actual
`ScissorAttrib` into the node's state to perform the GPU-level scissor;
when false, it culls only (frustum-narrowing) without scissoring pixels.

## Behavior notes

- Two modes, chosen at construction: **screen mode** (`make_screen()`) —
  a fixed `left,right,bottom,top` fraction, same as `ScissorAttrib`, no
  point projection needed each frame; **node mode** (`make_node()`) — 2 or
  4 (or more, via `add_point()`) points, each optionally relative to a
  *different* `NodePath` (empty `NodePath` = relative to the effect's own
  node), whose screen-space projections' bounding box becomes the scissor
  frame.
- `cull_callback()`: projects each point through `node_transform ∘
  modelview ∘ lens_projection`, takes the min/max in normalized device
  coords, remaps `-1..1 → 0..1`, then clamps into `[0,1]` (with `right ≥
  left` and `top ≥ bottom` enforced). Bails out early (does nothing) if the
  net transform is singular.
- The computed frame is also used to build an actual 3D bounding frustum
  (`make_frustum()`, via `Lens::extrude()` at near/far planes → a
  `BoundingHexahedron`) which is installed as `data._view_frustum` for the
  rest of the traversal under this node — this is the "performs culling"
  half of the effect, independent of whether `_clip` is set.
- `xform()`: node-relative point effects transform their untied (no
  explicit relative node) points by the given matrix; screen-mode and
  points with an explicit relative node are unaffected (`is_screen()`
  effects return `this` unchanged).
- Points and the frame are declared but the constructors are private —
  application code must go through `make_screen()`/`make_node()`/
  `add_point()`.

## API

| Method | Notes |
|---|---|
| `static CPT(RenderEffect) make_screen(const LVecBase4 &frame, bool clip = true)` | Fixed left/right/bottom/top fractions |
| `static CPT(RenderEffect) make_node(bool clip = true)` | Empty node-relative effect; add points via `add_point()` |
| `static CPT(RenderEffect) make_node(const LPoint3 &a, const LPoint3 &b, const NodePath &node = NodePath())` | 2-point (diagonal corners) form, `clip=true` |
| `static CPT(RenderEffect) make_node(const LPoint3 &a, const LPoint3 &b, const LPoint3 &c, const LPoint3 &d, const NodePath &node = NodePath())` | 4-point form |
| `CPT(RenderEffect) add_point(const LPoint3 &point, const NodePath &node = NodePath()) const` | Node-mode only; returns new effect |
| `bool is_screen() const` | |
| `const LVecBase4 &get_frame() const` | Screen mode only |
| `int get_num_points() const` / `const LPoint3 &get_point(int n) const` / `get_points()` | Node mode only |
| `NodePath get_node(int n) const` / `get_nodes()` | Per-point relative node, or empty = this node |
| `bool get_clip() const` | True = also emit ScissorAttrib; false = cull only |

## Usage

```cpp
// Node-relative scissor: only render (and only scissor-clip) the screen
// region covered by this node's bounding box corners.
model_np.set_effect(
  ScissorEffect::make_node(LPoint3(-1,-1,-1), LPoint3(1,1,1)));

// Cull-only variant (narrow the frustum but don't scissor pixels) — build
// via make_node(bool) + add_point(), since the point-taking overloads
// always clip.
CPT(RenderEffect) effect = ScissorEffect::make_node(false);
effect = DCAST(ScissorEffect, effect)->add_point(LPoint3(-1,-1,-1));
effect = DCAST(ScissorEffect, effect)->add_point(LPoint3(1,1,1));
model_np.set_effect(effect);
```

## See also

- [RenderEffect](RenderEffect.md) — base class
- [ScissorAttrib](ScissorAttrib.md) — the lower-level attrib this wraps
- [../pgraph/README.md](README.md) — cull pipeline overview
