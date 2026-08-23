# PolylightNode

**Source:** `panda/src/pgraph/polylightNode.h` (+ `.I`, `.cxx`)
**Inherits:** PandaNode

A non-realtime, non-hardware "poly light" — a point light approximation
applied by baking its contribution into vertex colors (via an external
effect/traverser, `PolylightNodeEffect`, not part of this module) rather
than through the GPU lighting pipeline used by [Light](Light.md). Useful
for large numbers of ambient-style lights where hardware light count
limits would otherwise apply.

## Behavior notes

- Distinct from [Light](Light.md)/[LightAttrib](LightAttrib.md) entirely —
  no `bind()`, no GSG interaction, no interning. This is a plain
  `PandaNode` carrying light-like data (`position`, `color`, `radius`,
  attenuation) that some external system reads and applies manually.
- **Attenuation**: `ALINEAR` or `AQUADRATIC`, computed as
  `fd = 1 / (a0 + a1*distance + a2*distance^2)`, tunable per-term via
  `set_a0()`/`set_a1()`/`set_a2()` (defaults `1.0, 0.1, 0.01`).
- **Flicker**: when `is_flickering()` (default `true`, `FRANDOM` type),
  `flicker()` perturbs the base color by a random or sinusoidal
  `variation` (`FSIN` uses `sinf(now * _sin_freq)`, clamped non-negative;
  `FRANDOM` uses `rand()%100 / 100.0`), scaled/offset by `_scale`/`_offset`
  — tuning knobs, not physically meaningful units. `FCUSTOM` is declared
  in the enum but unimplemented (comment: "Future addition").
  `polylight-info` config var enables debug logging of each computed
  variation.
- `get_color()` returns the node's own stored color; `get_color_scenegraph()`
  instead checks for a `ColorAttrib` already set on this node via
  `PandaNode::get_attrib()` and prefers that (if it's `T_flat`) — lets an
  external color-flatten operation on the node override the light's base
  color without the two mechanisms fighting.
- `xform()` transforms `_position` by the matrix and rescales `_radius`
  based on how a unit vector along X transforms — explicitly called out
  in the source as "cheesy" and wrong under non-uniform scale.
- `compare_to()`/`operator==`/`operator<` compare all light properties —
  intended for storing `PolylightNode`s in sorted STL containers (e.g. a
  distance-sorted active-lights list), not identity comparison.
- The old multi-parameter constructor is commented out in the header —
  interrogate (the Python-binding generator) would've generated an
  unwieldy wrapper for it, so construction is name-only and all other
  properties are set via `set_*()` calls.

## API

| Method | Notes |
|---|---|
| `PolylightNode(name)` | Constructor; enabled, white, radius 50, linear attenuation, random flicker on by default |
| `enable()` / `disable()` / `is_enabled()` | |
| `set_pos(...)` / `get_pos()` | |
| `set_color(...)` / `get_color()` / `get_color_scenegraph()` | Latter checks for an overriding flat `ColorAttrib` |
| `set_radius(float)` / `get_radius()` | Spherical volume of effect |
| `set_attenuation(Attenuation_Type)` / `get_attenuation()` | `ALINEAR` \| `AQUADRATIC` |
| `set_a0/a1/a2(float)` / `get_a0/a1/a2()` | Attenuation equation terms |
| `flicker_on()` / `flicker_off()` / `is_flickering()` | |
| `set_flicker_type(Flicker_Type)` / `get_flicker_type()` | `FRANDOM` \| `FSIN` (`FCUSTOM` unimplemented) |
| `set_offset/scale/step_size/freq(float)` + getters | Flicker-variation tuning |
| `flicker()` | Computes the current flicker-varied color |
| `compare_to(other)` / `==` / `!=` / `<` | Full-property comparison, for sorted containers |

## See also

- [Light](Light.md) — the hardware-lighting counterpart; unrelated API, similar role
- [PolylightEffect](PolylightEffect.md) — RenderEffect that likely drives PolylightNode application
