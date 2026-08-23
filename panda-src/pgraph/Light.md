# Light

**Source:** `panda/src/pgraph/light.h` (+ `.I`, `.cxx`)
**Not a PandaNode subclass** — a mixin interface, multiply-inherited by
concrete light node classes. **Implemented by:** `AmbientLight`,
`PointLight`, `DirectionalLight`, `Spotlight`, `SphereLight`, `RectangleLight`
(all in `panda/src/pgraphnodes`, undocumented in this reference).

The abstract interface to all kinds of lights. A concrete light class
inherits from *both* `Light` and `PandaNode` (e.g.
`class PointLight : public Light, public LensNode`-ish pattern in
`pgraphnodes`), so a light can be parented into the scene graph like any
other node — its position in the graph defines its coordinate space of
effect — while still being usable polymorphically as a `Light` (e.g. when
attached to a [`LightAttrib`](LightAttrib.md)).

## Behavior notes

- `as_node()` is the pure-virtual escape hatch back to `PandaNode*` — since
  `Light` doesn't inherit `PandaNode` itself, code holding a `Light*` needs
  this to get back to the node (e.g. to build a `NodePath`).
- **Color vs. color temperature**: a light's color can be set directly
  (`set_color()`, which also clears `_has_color_temperature`) or via
  `set_color_temperature(kelvins)`, which converts a blackbody temperature
  to sRGB via the CIE xyY color-space approximation and stores the result
  as `_color`. Default is 6500K (D65 white point, `x=0.31271, y=0.32902`
  used as a shortcut instead of running the polynomial fit). Both are
  mutually exclusive — setting one clears the other's "is set" flag, but
  `_color` is always kept in sync so `get_color()` works regardless of
  which was used.
- **Priority and sorting**: `set_priority()` bumps a global
  `_sort_seq` (`UpdateSeq`, shared across all `Light` instances) so that
  every `LightAttrib` in the world knows to re-sort its cached light list.
  `get_class_priority()` (pure virtual, one per concrete light type — see
  the `ClassPriority` enum: ambient < point < directional < spot < area)
  is the tiebreaker when two lights share the same explicit `set_priority()`
  value; the highest-priority *n* lights are the ones actually bound to
  hardware when more lights are active than the GSG supports.
- `attrib_ref()`/`attrib_unref()` are called when a light is added to /
  removed from a `LightAttrib` — base implementation is a no-op; a
  subclass could use these to track "how many LightAttribs reference me."
- `get_vector_to_light()` is a default `return false` (meaningless for an
  ambient light); concrete directional/point light types override it to
  compute the actual to-light vector for shading math.
- `get_viz()` lazily builds (and caches, via `_viz_geom_stale`) a
  `GeomNode` representing the light for visual debugging, rebuilt only
  when `mark_viz_stale()` was called since the last fetch — used during
  cull to render visible lights (`show-lights`-style debug tooling).
- The `CData` (cycled) block holds only `_color` and the cached viz geom —
  `_priority`, `_has_color_temperature`, `_color_temperature` are
  deliberately *not* pipeline-cycled ("no real reason to... makes it
  difficult to synchronize with the LightAttribs").

## API

| Method | Notes |
|---|---|
| `as_node()` = 0 | Pure virtual; returns the `PandaNode*` this Light also is |
| `is_ambient_light()` | `false` by default; `AmbientLight` overrides to `true` |
| `get_color()` / `set_color(LColor)` | Direct color; clears color-temperature flag |
| `has_color_temperature()` / `get_color_temperature()` / `set_color_temperature(K)` | Temperature-based color (since 1.10.0) |
| `get_exponent()` | Spotlight falloff exponent; `0` for non-spot lights |
| `get_specular_color()` | Meaningless for ambient lights; default white |
| `get_attenuation()` | `(constant, linear, quadratic)` distance falloff terms; default `(1,0,0)` = no falloff |
| `set_priority(int)` / `get_priority()` | Explicit relative importance |
| `get_class_priority()` = 0 | Per-subclass tiebreaker (ambient < point < directional < spot < area) |
| `get_sort_seq()` (static) | Global `UpdateSeq`, bumped on any priority change, watched by `LightAttrib` |
| `bind(gsg, NodePath, light_id)` = 0 | Binds this light to a GSG hardware light slot |
| `get_viz()` | Lazily-built debug-visualization `GeomNode` |

## See also

- [LightAttrib](LightAttrib.md) — collects `Light*`s (via their `PandaNode`
  identity) into the state pipeline; calls `attrib_ref()`/`attrib_unref()`
  and watches `get_sort_seq()`
- [GeomNode](GeomNode.md) — type returned by `get_viz()`
