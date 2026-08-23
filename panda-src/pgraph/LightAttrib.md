# LightAttrib

**Source:** `panda/src/pgraph/lightAttrib.h` (+ `.I`, `.cxx`)
**Inherits:** RenderAttrib

Controls which set of `Light`-casting nodes illuminate geometry at and
below this node. Unlike most `RenderAttrib`s, it carries two independent
lists — `on_lights` and `off_lights` — plus an optional "off all" flag, so
composing two `LightAttrib`s down the scene graph merges/overrides the
active light set incrementally rather than one fully replacing the other.
`Light` here is [the mixin interface class](README.md#class-hierarchy)
(`panda/src/pgraph/light.h`) implemented by concrete light node types
(PointLight, DirectionalLight, Spotlight, AmbientLight, …) defined in
`panda/src/pgraphnodes` (undocumented); lights are always referenced by
`NodePath`, not `Light*`, in the modern API.

## Behavior notes

- **Deprecated `Operation`-based interface** (`make(op, light...)`,
  `get_operation()`, `get_num_lights()`/`get_light()`/`has_light()`,
  `add_light()`/`remove_light()`) is legacy — it logs an `pgraph_cat`
  warning on every call and internally maps onto the modern
  `on_lights`/`off_lights` interface. New code should use
  `add_on_light()`/`add_off_light()`/etc. directly.
- `make()` and `make_all_off()` each memoize a single shared empty/all-off
  instance in a static `CPT(RenderAttrib)` on first call — cheaper than
  relying on interning lookup for these very common cases.
- `add_on_light()` also removes the light from `off_lights` if present, and
  vice versa via `add_off_light()` — the two lists are kept mutually
  exclusive.
- Lights are ref-counted specially: `attrib_ref()`/`attrib_unref()` on the
  `Light` interface are called whenever a light enters/leaves the
  `on_lights` set (copy constructor, destructor, `add_on_light()`,
  `remove_on_light()`) — separate from the normal `NodePath`/`PandaNode`
  refcount, used by lights to track how many attribs reference them.
- `on_lights` is lazily sorted by priority (`check_sorted()` compares a
  cached `UpdateSeq` against `Light::get_sort_seq()`, a global sequence
  bumped whenever any light's priority changes) — non-ambient lights sort
  first by `get_priority()` then `get_class_priority()`, descending;
  ambient lights are appended last regardless of priority since their
  contribution is just summed (`get_ambient_contribution()`), not ordered.
  `get_most_important_light()` returns the first non-ambient light after
  sorting, or an empty `NodePath` if none.
- `compose_impl()`: if the next attrib turns off all lights, it wins
  outright. Otherwise it's a 3-way sorted merge of this attrib's
  `on_lights` against the next attrib's `on_lights` and `off_lights` — a
  light on in both stays on; a light on here but turned off downstream is
  dropped; a light newly turned on downstream is added. `invert_compose_impl()`
  always just returns the other attrib (not meaningfully invertible).
- Bam I/O: pre-file-version-40 files stored raw `PandaNode*` pointers
  instead of `NodePath`s, resolved to `NodePath`s in `finalize()` via
  `AttribNodeRegistry` lookup (by type+name) — a versioned compatibility
  path, not something new code needs to think about.

## API

| Signature | Notes |
|---|---|
| `static CPT(RenderAttrib) make()` | Identity attrib — no lights on or off |
| `static CPT(RenderAttrib) make_all_off()` | Disables all lighting |
| `static CPT(RenderAttrib) make_default()` | Same as `make()` |
| `size_t get_num_on_lights() const` / `NodePath get_on_light(size_t n) const` | Lights turned on, sorted by priority (ambients last) |
| `MAKE_SEQ(get_on_lights, ...)` | Python-style sequence accessor |
| `size_t get_num_non_ambient_lights() const` | Count of non-ambient on-lights (they sort first) |
| `bool has_on_light(const NodePath &) const` / `has_any_on_light() const` | Membership tests |
| `size_t get_num_off_lights() const` / `NodePath get_off_light(size_t n) const` | Lights explicitly turned off (arbitrary order) |
| `bool has_off_light(const NodePath &) const` | True if in off list, or `has_all_off()` and not in on list |
| `bool has_all_off() const` | True if this attrib turns off all lights |
| `bool is_identity() const` | True if attrib changes nothing |
| `CPT(RenderAttrib) add_on_light/remove_on_light/replace_on_light(...)` | Returns a new attrib with the on-list modified |
| `CPT(RenderAttrib) add_off_light/remove_off_light/replace_off_light(...)` | Returns a new attrib with the off-list modified |
| `NodePath get_most_important_light() const` | Highest-priority non-ambient on-light, or empty NodePath |
| `LColor get_ambient_contribution() const` | Sum of all ambient on-lights' colors |
| `MAKE_SEQ_PROPERTY(on_lights, ...)` / `MAKE_SEQ_PROPERTY(off_lights, ...)` | Property-style sequence access |

Deprecated: `enum Operation {O_set, O_add, O_remove}`, `make(op, light1[, light2[, light3[, light4]]])`,
`get_operation()`, `get_num_lights()`, `get_light(n)`, `has_light()`,
`add_light()`, `remove_light()`.

## Usage

```cpp
NodePath light_np = render.attach_new_node(new DirectionalLight("sun"));
node_path.set_light(light_np);          // NodePath::set_light() wraps add_on_light()
node_path.set_light_off(other_light_np); // wraps add_off_light()

// Direct RenderAttrib form:
CPT(RenderAttrib) la = LightAttrib::make();
la = DCAST(LightAttrib, la)->add_on_light(light_np);
node_path.set_attrib(la);
```

## See also

[README — the state pipeline](README.md#the-state-pipeline),
[RenderAttrib](RenderAttrib.md)
