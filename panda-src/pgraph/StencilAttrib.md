# StencilAttrib

**Source:** `panda/src/pgraph/stencilAttrib.h` (+ `.I`, `.cxx`)
**Inherits:** RenderAttrib

Holds the full set of stencil-buffer render states (comparison function,
pass/fail operations, reference value, read/write masks — separately for
front and back faces) as one read-only bundle. Query
`GraphicsStateGuardian::get_supports_two_sided_stencil()` before relying on
independent front/back stencil ops.

## Behavior notes

- The default constructor sets both front and back comparison functions to
  `M_none` (stenciling disabled) and all operations to `SO_keep`.
- `make()` and `make_with_clear()` build front-face-only attribs (back face
  forced to `M_none`/`SO_keep`); `make_2_sided()`/`make_2_sided_with_clear()`
  set both faces independently. If `front_enable`/`back_enable` is false,
  that face's comparison function is forced to `M_none` regardless of what
  was passed.
- `PandaCompareFunc` (`M_never`...`M_always`) is the same enum used by
  `DepthTestAttrib`/`AlphaTestAttrib`; `StencilComparisonFunction`
  (`SCF_*`) is a deprecated alias set kept "purely for backward
  compatibility."
- `compare_to_impl()`/`get_hash_impl()` iterate all `SRS_total` (13)
  states — no shortcuts, no `compose_impl()` override (default replace
  semantics).
- Bam I/O has a version split at minor version 35: older files stored
  front/back enable as separate bools plus comparison functions offset by
  one (with `7` as a disabled sentinel); version ≥35 stores all 13 states
  directly. Not relevant to new code, just a compatibility detail.

## API

| Signature | Notes |
|---|---|
| `enum StencilRenderState` | `SRS_front_comparison_function`, `SRS_front_stencil_fail_operation`, `SRS_front_stencil_pass_z_fail_operation`, `SRS_front_stencil_pass_z_pass_operation`, `SRS_reference`, `SRS_read_mask`, `SRS_write_mask`, `SRS_back_*` (mirrors front), `SRS_clear`, `SRS_clear_value` |
| `enum StencilOperation` | `SO_keep`, `SO_zero`, `SO_replace`, `SO_increment`, `SO_decrement`, `SO_invert`, `SO_increment_saturate`, `SO_decrement_saturate` |
| `static CPT(RenderAttrib) make_off()` / `make_default()` | Stenciling disabled |
| `static CPT(RenderAttrib) make(front_enable, front_comparison_function, stencil_fail_operation, stencil_pass_z_fail_operation, front_stencil_pass_z_pass_operation, reference, read_mask, write_mask=~0)` | Front-face only |
| `static CPT(RenderAttrib) make_2_sided(...)` | Independent front + back state |
| `static CPT(RenderAttrib) make_with_clear(..., bool clear, unsigned int clear_value)` | Front-face + auto-clear |
| `static CPT(RenderAttrib) make_2_sided_with_clear(...)` | Two-sided + auto-clear |
| `unsigned int get_render_state(StencilRenderState id) const` | Query any single state by enum |

## Usage

```cpp
node_path.set_attrib(StencilAttrib::make(
    true, RenderAttrib::M_always,
    StencilAttrib::SO_keep, StencilAttrib::SO_keep, StencilAttrib::SO_replace,
    1, 0xff, 0xff));
```

## See also

[README — the state pipeline](README.md#the-state-pipeline),
[RenderAttrib](RenderAttrib.md), [DepthTestAttrib](DepthTestAttrib.md)
