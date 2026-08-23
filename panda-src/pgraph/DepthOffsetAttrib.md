# DepthOffsetAttrib

**Source:** `panda/src/pgraph/depthOffsetAttrib.h` (+ `.I`, `.cxx`)
**Inherits:** [RenderAttrib](RenderAttrib.md)

Instructs the graphics driver to bias generated depth values before
they're written to the depth buffer — used to shift polygons slightly
toward the camera to resolve z-fighting (e.g. decals). Driver support is
described as "spotty" in the source, so use with caution.

## Behavior notes
- The bias is an integer; each increment is the smallest Z step needed to
  fully resolve two coplanar polygons. Positive = closer to camera.
- **Offsets accumulate through nesting**: a `DepthOffsetAttrib(1)` beneath
  a `DepthOffsetAttrib(2)` composes to a net offset of 3 —
  `compose_impl()` adds `_offset + other->_offset` (and carries forward
  `other`'s min/max range, not this one's). `invert_compose_impl()`
  subtracts instead.  A `DepthOffsetAttrib` will *not* compose across a
  lower attrib with a higher override value (standard `RenderAttrib`
  override behavior, not special-cased here). Net accumulated value should
  stay roughly within 0–16 for driver portability (source comment).
- Tangential second feature, unrelated to the bias: `min_value`/`max_value`
  (both in `[0, 1]`, default `0, 1`) constrain (or reverse, if inverted)
  the output Z range written to the depth buffer, for custom depth-buffer
  tricks.
- Bam files before minor version 31 didn't store min/max — they default to
  `0.0, 1.0` on load from old files.

## API

| Method | Notes |
|---|---|
| `static CPT(RenderAttrib) make(int offset = 1)` | min/max default to 0/1 |
| `static CPT(RenderAttrib) make(int offset, PN_stdfloat min_value, PN_stdfloat max_value)` | Both in `[0,1]` |
| `static CPT(RenderAttrib) make_default()` | Offset 0, range [0,1] |
| `int get_offset() const` | |
| `PN_stdfloat get_min_value() const` / `get_max_value() const` | |

## Usage
```cpp
// Push decal geometry 1 unit closer to avoid z-fighting with its base surface
decal_np.set_attrib(DepthOffsetAttrib::make(1));
```

## See also
[RenderAttrib](RenderAttrib.md), [DepthTestAttrib](DepthTestAttrib.md),
[pgraph README — the state pipeline](README.md#the-state-pipeline)
