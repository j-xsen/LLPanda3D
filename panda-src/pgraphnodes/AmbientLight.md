# AmbientLight

**Source:** `panda/src/pgraphnodes/ambientLight.h` (+ `.I`, `.cxx`)
**Inherits:** [LightNode](LightNode.md) > `Light`, `PandaNode`

A light source that illuminates every point in the scene equally,
regardless of position or surface orientation — no direction, no falloff,
only a flat color contribution added everywhere `LightAttrib` enables it.
Because it has no meaningful position, an `AmbientLight` doesn't
technically need to be parented into the scene graph at all (though it
usually is, for consistency with other lights and to control its
color/priority through the graph like anything else).

## Behavior notes

- **`bind()` deliberately raises an assertion** (`nassert_raise("cannot
  bind AmbientLight")`) rather than doing anything — ambient light has no
  per-light GPU binding slot the way positional/directional lights do
  (`GraphicsStateGuardian::bind_light()`); its color is instead summed into
  the overall scene ambient term. This assertion firing indicates a bug in
  the light-binding code attempting to treat an `AmbientLight` like a
  bindable light.
- **`get_class_priority()` returns the lowest of all light types**
  (`CP_ambient_priority`) — used only to break ties when two lights have
  equal explicit `get_priority()`; all else equal, ambient lights are
  considered least important; overridden and increasing per light type
  (directional > point > spot > area, based on each light's
  `get_class_priority()` override).

## API

| Method | Notes |
|---|---|
| `AmbientLight(name)` | Constructor |
| `get_class_priority()` | Virtual override; returns `CP_ambient_priority` |
| `is_ambient_light()` | `final`; returns `true` — faster than `is_of_type()` |

Color get/set is inherited unchanged from `Light`
(see [../pgraph/Light.md](../pgraph/Light.md)).

## Usage

```cpp
NodePath render = window->get_render();  // WindowFramework* window
PT(AmbientLight) alight = new AmbientLight("ambient");
alight->set_color(LColor(0.2f, 0.2f, 0.2f, 1.0f));
NodePath alnp = render.attach_new_node(alight);
render.set_light(alnp);
```

## See also

- [LightNode](LightNode.md) — base class
- [../pgraph/Light.md](../pgraph/Light.md) — shared `Light` interface, `LightAttrib`
