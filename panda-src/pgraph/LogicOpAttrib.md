# LogicOpAttrib

**Source:** `panda/src/pgraph/logicOpAttrib.h` (+ `.I`, `.cxx`)
**Inherits:** RenderAttrib

Since 1.10.0. If enabled (any value other than `O_none`), replaces normal
color blending with a raw bitwise logical operation between the incoming
fragment color and the framebuffer's existing color.

## Behavior notes

- Setting a logic op other than `O_none` disables color blending entirely
  (per the class doc comment) — don't combine with `ColorBlendAttrib` for a
  blended effect, the logic op wins.
- `make_off()` and `make_default()` both return the slot's registered
  default (`O_none`) via `RenderAttribRegistry::quick_get_global_ptr()->
  get_slot_default(_attrib_slot)` rather than constructing a fresh
  instance — they're equivalent.
- `operator<<(ostream&, Operation)` gives each enum value's OpenGL-style
  name (`"and_reverse"`, `"xor"`, `"nand"`, etc.) — useful for debug output.

## API

| Signature | Notes |
|---|---|
| `enum Operation` | `O_none` (disabled/normal blending), `O_clear`, `O_and`, `O_and_reverse`, `O_copy`, `O_and_inverted`, `O_noop`, `O_xor`, `O_or`, `O_nor`, `O_equivalent`, `O_invert`, `O_or_reverse`, `O_copy_inverted`, `O_or_inverted`, `O_nand`, `O_set` |
| `static CPT(RenderAttrib) make(Operation op)` | |
| `static CPT(RenderAttrib) make_off()` | Equivalent to `make(O_none)` |
| `static CPT(RenderAttrib) make_default()` | Same as `make_off()` |
| `Operation get_operation() const` | |

## Usage

```cpp
node_path.set_attrib(LogicOpAttrib::make(LogicOpAttrib::O_xor));
// ...
node_path.set_attrib(LogicOpAttrib::make_off());
```

## See also

[README — the state pipeline](README.md#the-state-pipeline),
[RenderAttrib](RenderAttrib.md), [ColorBlendAttrib](ColorBlendAttrib.md)
