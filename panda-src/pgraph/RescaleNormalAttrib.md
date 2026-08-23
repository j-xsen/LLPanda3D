# RescaleNormalAttrib

**Source:** `panda/src/pgraph/rescaleNormalAttrib.h` (+ `.I`, `.cxx`)
**Inherits:** RenderAttrib

Controls whether/how surface normals are adjusted at render time to
compensate for scale transforms in the scene graph, so lighting stays
correct even under a scaled `NodePath`.

## Behavior notes

- Only 4 possible instances ever exist — `make()` indexes a static
  `_attribs[M_auto + 1]` array by `Mode` instead of going through the usual
  interning registry ("the only way a RescaleNormalAttrib can be made
  anyway," per the source comment).
- `make_default()` returns the top-of-graph default driven by the
  `rescale-normals` config variable (default `M_auto`), read once in
  `init_type()` — not a hardcoded default like most other attribs'
  `make_default()`.
- `M_rescale` counterscales normals by the transform's uniform scale
  (cheap, GPU-supported); `M_normalize` renormalizes to unit length
  (correct for non-uniform scale, potentially expensive); `M_auto` picks
  rescale under uniform scale and normalize under non-uniform scale.
- Has `operator>>` (`std::istream`) in addition to `operator<<`, parsing
  the mode name case-insensitively — used by the `ConfigVariableEnum`
  machinery for `rescale-normals`.

## API

| Signature | Notes |
|---|---|
| `enum Mode` | `M_none`, `M_rescale`, `M_normalize`, `M_auto` |
| `static CPT(RenderAttrib) make(Mode mode)` | |
| `static CPT(RenderAttrib) make_default()` | Driven by `rescale-normals` config var |
| `Mode get_mode() const` | |

## Usage

```cpp
render.set_attrib(RescaleNormalAttrib::make(RescaleNormalAttrib::M_auto));
```

## See also

[README — the state pipeline](README.md#the-state-pipeline),
[RenderAttrib](RenderAttrib.md)
