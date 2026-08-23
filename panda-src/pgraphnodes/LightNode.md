# LightNode

**Source:** `panda/src/pgraphnodes/lightNode.h` (+ `.I`, `.cxx`)
**Inherits:** `Light` (external, [../pgraph/Light.md](../pgraph/Light.md)), `PandaNode` **Inherited by:** [AmbientLight](AmbientLight.md)

Base class for lights that need no frustum/direction/position — a plain
`PandaNode` combined with the abstract `Light` interface. Of the six
concrete lights in this module, only [AmbientLight](AmbientLight.md) uses
this base directly; every other light needs a direction and/or shape, so it
inherits [LightLensNode](LightLensNode.md) instead (see the
[pgraphnodes README](README.md#lights-lightnode-vs-lightlensnode) for why
two bases exist).

## Behavior notes

- **Multiple inheritance from `Light` and `PandaNode` requires explicit
  disambiguation.** `output()`/`write()` are re-declared `PUBLISHED` here
  purely to resolve which base's version gets exposed to the scripting
  layer — both bases declare them, and C++ needs the derived class to pick
  one (delegated straight to `PandaNode::output()`/`write()`).
- **`as_node()`/`as_light()` both just `return this`** — they exist so code
  holding a generic `Light*` or `PandaNode*` can safely cross-cast to the
  other interface without a fragile `dynamic_cast` through the diamond,
  since the compiler can't implicitly convert between unrelated base
  pointers of a multiply-inherited object.
- Persistence combines `PandaNode::write_datagram()`/`fillin()` and
  `Light::write_datagram()`/`fillin()` — two independent datagram sections,
  same pattern used by [LightLensNode](LightLensNode.md).

## API

Adds no new public API beyond what `Light` and `PandaNode` already provide
— this class exists purely to combine them. See
[../pgraph/Light.md](../pgraph/Light.md) for the shared light API
(`get_color()`/`set_color()`, `get_priority()`, etc.) common to every light
in this module.

| Method | Notes |
|---|---|
| `LightNode(name)` | Constructor |
| `as_node()` / `as_light()` | Cross-cast helpers, both just `return this` |

## See also

- [AmbientLight](AmbientLight.md) — the one concrete subclass
- [LightLensNode](LightLensNode.md) — sibling base for every other light
- [../pgraph/Light.md](../pgraph/Light.md) — the shared abstract interface
