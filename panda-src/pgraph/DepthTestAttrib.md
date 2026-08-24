# DepthTestAttrib

**Source:** `panda/src/pgraph/depthTestAttrib.h` (+ `.I`, `.cxx`)
**Inherits:** [RenderAttrib](RenderAttrib.md)

Controls the depth-buffer comparison function used to decide whether a
newly rendered pixel is kept (its depth passes the test against the
existing buffer value). Not to be confused with
[DepthWriteAttrib](DepthWriteAttrib.md), which controls whether depth
values are *written* — the header's doc comment on this class is a
copy-paste leftover ("enables or disables writing to the depth buffer")
and actually describes `DepthWriteAttrib`; this class is the *test*
function.

## Behavior notes
- Uses the same shared `PandaCompareFunc` enum as
  [AlphaTestAttrib](AlphaTestAttrib.md) (`M_never` .. `M_always`).
- `make_default()`'s implicit constructor default is `M_less` — standard
  "closer wins" depth testing.
- No custom `compose_impl` — a later attrib in the graph replaces the
  earlier one directly (default `RenderAttrib` behavior).

## API

| Method | Notes |
|---|---|
| `static CPT(RenderAttrib) make(PandaCompareFunc mode)` | |
| `static CPT(RenderAttrib) make_default()` | `M_less` |
| `PandaCompareFunc get_mode() const` | |

## Usage
```cpp
node_path.set_attrib(DepthTestAttrib::make(DepthTestAttrib::M_always));  // disable depth testing
```

## See also
[RenderAttrib](RenderAttrib.md), [DepthWriteAttrib](DepthWriteAttrib.md),
[AlphaTestAttrib](AlphaTestAttrib.md), [pgraph README — the state pipeline](README.md#the-state-pipeline)
