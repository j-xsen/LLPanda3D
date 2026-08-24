# CullFaceAttrib

**Source:** `panda/src/pgraph/cullFaceAttrib.h` (+ `.I`, `.cxx`)
**Inherits:** [RenderAttrib](RenderAttrib.md)

Indicates which faces (front/back, by vertex winding order) should be
backface-culled. Panda convention: vertices wind counterclockwise when
viewed from the front, so `M_cull_clockwise` culls backfaces.

## Behavior notes
- `_mode` (actual, as set) vs. `get_effective_mode()` (actual mode with
  the `_reverse` flag applied) are distinct: `make_reverse()` builds an
  attrib with mode `M_cull_unchanged` and `reverse = true`, which flips
  whatever culling sense is otherwise in effect (CW↔CCW) without itself
  specifying a mode — useful for e.g. a mirrored/reflected sub-scene where
  winding order effectively inverts.
- `M_cull_unchanged` is the identity mode: composing it with a preceding
  attrib keeps the preceding mode (see `compose_impl`).
- `compose_impl`: in the common case (`this` isn't reversing and `other`
  specifies a real mode), `other` replaces `this` directly. Otherwise mode
  and reverse-ness combine: `other`'s mode wins if it's not
  `M_cull_unchanged`, else `this`'s mode carries through; the two
  `_reverse` flags XOR together. `invert_compose_impl` mirrors this with
  `_reverse`'s meaning inverted.
- `get_effective_mode()`: without reverse, `M_cull_clockwise`/
  `M_cull_unchanged` → `M_cull_clockwise`, `M_cull_counter_clockwise` →
  itself; with reverse, CW and CCW swap. Any other mode (i.e.
  `M_cull_none`) maps to `M_cull_none` regardless of reverse.

## API

| Method | Notes |
|---|---|
| `static CPT(RenderAttrib) make(Mode mode = M_cull_clockwise)` | |
| `static CPT(RenderAttrib) make_reverse()` | `M_cull_unchanged`, reverse=true; flips CW/CCW sense |
| `static CPT(RenderAttrib) make_default()` | `M_cull_clockwise`, reverse=false |
| `Mode get_actual_mode() const` | Ignores reverse flag |
| `bool get_reverse() const` | |
| `Mode get_effective_mode() const` | Actual mode with reverse applied |

`Mode` enum: `M_cull_none`, `M_cull_clockwise`, `M_cull_counter_clockwise`,
`M_cull_unchanged` (identity — doesn't change inherited cull behavior).

## Usage
```cpp
node_path.set_attrib(CullFaceAttrib::make(CullFaceAttrib::M_cull_none));  // disable backface culling
```

## See also
[RenderAttrib](RenderAttrib.md), [pgraph README — the state pipeline](README.md#the-state-pipeline)
