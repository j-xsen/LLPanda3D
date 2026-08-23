# DepthWriteAttrib

**Source:** `panda/src/pgraph/depthWriteAttrib.h` (+ `.I`, `.cxx`)
**Inherits:** [RenderAttrib](RenderAttrib.md)

Enables or disables writing to the depth buffer (as opposed to
[DepthTestAttrib](DepthTestAttrib.md), which controls whether existing
depth values are *tested against*). Useful for rendering geometry that
should be depth-tested but not itself occlude later geometry (e.g.
transparent overlays).

## Behavior notes
- `make_default()`'s implicit constructor default is `M_on` — depth
  writing enabled is the default state.
- No custom `compose_impl` — later attrib replaces the earlier one.

## API

| Method | Notes |
|---|---|
| `static CPT(RenderAttrib) make(Mode mode)` | |
| `static CPT(RenderAttrib) make_default()` | `M_on` |
| `Mode get_mode() const` | |

`Mode` enum: `M_off`, `M_on`.

## Usage
```cpp
node_path.set_attrib(DepthWriteAttrib::make(DepthWriteAttrib::M_off));
```

## See also
[RenderAttrib](RenderAttrib.md), [DepthTestAttrib](DepthTestAttrib.md),
[pgraph README — the state pipeline](README.md#the-state-pipeline)
