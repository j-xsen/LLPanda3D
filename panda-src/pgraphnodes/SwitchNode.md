# SwitchNode

**Source:** `panda/src/pgraphnodes/switchNode.h` (+ `.I`, `.cxx`)
**Inherits:** [SelectiveChildNode](SelectiveChildNode.md) > `PandaNode`

Renders exactly one of its children, chosen explicitly by index via
`set_visible_child()` — no distance or time computation involved, unlike
[LODNode](LODNode.md)/[SequenceNode](SequenceNode.md). Useful for manual
show/hide toggling between mutually-exclusive alternatives (e.g. damaged
vs. undamaged model variants) where the application, not the engine,
decides which is current.

## Behavior notes

- **The visible-child index lives in a pipeline-cycled `CData`**, unlike
  `SelectiveChildNode`'s own `_selected_child` — `SwitchNode` doesn't
  actually use the base class's `_selected_child`/`select_child()`
  machinery for storage, only for the `has_selective_visibility()`
  traversal protocol; `cull_callback()` calls
  `select_child(get_visible_child())` each frame to sync the two.
- **`has_single_child_visibility()` returns `true`** — a marker (also true
  for [SequenceNode](SequenceNode.md)) meaning the visible child can be
  determined without any external context (no camera/distance needed),
  letting callers use `get_visible_child()` directly instead of the more
  general first/next-visible-child protocol.
- **`safe_to_combine()`/`safe_to_combine_children()` are both `false`** —
  same reasoning as `LODNode`: child order/identity is semantically
  meaningful, so scene graph flattening must not merge them.
- An out-of-range `set_visible_child()` index isn't validated at set time —
  it's compared against `get_num_children()` at cull time via the
  inherited `has_selective_visibility()` protocol, so setting an invalid
  index results in nothing being rendered rather than an error.

## API

| Method | Notes |
|---|---|
| `SwitchNode(name)` | Constructor |
| `set_visible_child(index)` / `get_visible_child()` | The single child index rendered; `MAKE_PROPERTY` |

## Usage

```cpp
PT(SwitchNode) sw = new SwitchNode("damage_states");
sw->add_child(undamaged_model);  // index 0
sw->add_child(damaged_model);    // index 1
sw->set_visible_child(0);
```

## See also

- [SelectiveChildNode](SelectiveChildNode.md) — base class
- [SequenceNode](SequenceNode.md) — time-driven sibling
- [LODNode](LODNode.md) — distance-driven "one visible child" alternative
