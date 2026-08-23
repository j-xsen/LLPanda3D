# UvScrollNode

**Source:** `panda/src/pgraphnodes/uvScrollNode.h` (+ `.I`, `.cxx`)
**Inherits:** `PandaNode`
**Inherited by:** (none)

A node placed at a key point in the scene graph to continuously animate
texture coordinates on everything below it — scrolling, tiling, and/or
rotating a texture over time (e.g. flowing water, a conveyor belt, a
scrolling energy-shield effect). It affects only the **default texture
stage** (`TextureStage::get_default()`); it does not support animating
UVs for a specific named stage.

## Behavior notes

- **Computed entirely at cull time from wall-clock elapsed time — not a
  task.** `cull_callback()` computes `elapsed = current_frame_time -
  _start_time` and derives a fresh `TransformState` each time the node is
  culled: `(u*elapsed mod 1, v*elapsed mod 1, w*elapsed mod 1)` for
  translation and `(r*elapsed*360 mod 360, 0, 0)` for the HPR heading
  component. There's no per-frame `AsyncTask` and no stored "current
  offset" — the transform is a pure function of elapsed time since
  construction, recomputed on every cull traversal. This means: (1) the
  animation is deterministic and identical across multiple cameras/passes
  in the same frame, and (2) it keeps animating even if nothing renders it
  for a while — resuming visibility jumps straight to the "correct" phase
  for however much time has passed, it never falls behind and catches up.
- **Implemented as a composed `TexMatrixAttrib`, not by mutating any
  `Texture`.** The computed transform is wrapped in
  `TexMatrixAttrib::make(TextureStage::get_default(), ts)` (see
  [../pgraph/TexMatrixAttrib.md](../pgraph/TexMatrixAttrib.md)) and
  composed onto `data._state` for everything below this node in the
  traversal — so it's a cheap, GPU-side texture-coordinate transform
  (equivalent to setting a texture matrix), not a CPU-side UV rewrite of
  any `GeomVertexData`.
- `_start_time` is captured once at construction
  (`ClockObject::get_global_clock()->get_frame_time()`), not reset by
  `set_*_speed()` — changing a speed does not restart the phase, it just
  changes the rate from that point in elapsed time forward (which can
  produce a visible jump if the sign flips, since phase continuity is
  preserved but the accumulated distance traveled is not re-zeroed).
- `safe_to_flatten()` and `safe_to_combine()` both return `false` — like
  the other special-purpose nodes in this module, an individual
  `UvScrollNode`'s identity/position in the graph is meaningful and must
  survive scene graph flattening.
- Bam versioning: `_w_speed` is only read/written for files at minor
  version ≥ 33, `_r_speed` only for minor version ≥ 22 — loading an older
  `.bam` leaves those fields at their default-constructed value (0).

## API

| Method | Notes |
|---|---|
| `UvScrollNode(const std::string &name, PN_stdfloat u_speed, v_speed, w_speed, r_speed)` | Construct with per-second scroll rates for U, V, W (3rd texcoord dimension) and a rotation rate (degrees/sec, via heading). |
| `UvScrollNode(const std::string &name)` | Construct with all rates zero. |
| `set_u_speed`/`set_v_speed`/`set_w_speed`/`set_r_speed` + getters | Adjust rates at runtime; does not reset phase (see Behavior notes). |
| `u_speed`/`v_speed`/`w_speed`/`r_speed` (`MAKE_PROPERTY`) | Property-style accessors. |

## Usage

```cpp
PT(UvScrollNode) scroll = new UvScrollNode("water_scroll", 0.1, 0.05, 0, 0);
NodePath np = water_geom.attach_new_node(scroll);
// UVs on everything parented under `np` scroll continuously.
```

## See also

- [../gobj/TextureStage.md](../gobj/TextureStage.md) — the default stage
  this node affects
- [../pgraph/TexMatrixAttrib.md](../pgraph/TexMatrixAttrib.md) — the
  `RenderAttrib` this node composes onto the scene graph each frame
