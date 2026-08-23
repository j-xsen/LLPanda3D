# RenderEffects

**Source:** `panda/src/pgraph/renderEffects.h` (+ `.I`, `.cxx`)
**Inherits:** TypedWritableReferenceCount

The immutable, interned set-of-[RenderEffect](RenderEffect.md) container
held by every [PandaNode](PandaNode.md), analogous to how
[RenderState](RenderState.md) holds a set of `RenderAttrib`s. Unlike
`RenderState`, effects have no slot/override system — it's a simple
type-keyed set (`ov_set<Effect>`, ordered by `TypeHandle`), since effects
don't compose down the graph, they just apply once at the node that has
them. Constructed via `make()`/`add_effect()`/`remove_effect()`, never
mutated in place.

## Behavior notes

- `has_decal()`, `has_show_bounds()`, `has_show_tight_bounds()`,
  `has_cull_callback()`, `has_adjust_transform()` are all lazily-computed
  and cached (`_flags` bits like `F_checked_decal`/`F_has_decal`) by
  scanning the contained effects on first query — this is how `PandaNode`
  and the cull traverser cheaply check "does this node need decal handling
  / a cull callback / etc." without iterating the effect set every frame.
- `safe_to_transform()`/`safe_to_combine()` are the AND of every contained
  effect's answer — one effect voting "not safe" makes the whole set unsafe,
  which is what stops `NodePath::flatten_strong()` from touching nodes
  carrying things like `CompassEffect`.
- `cull_callback()`/`adjust_transform()` dispatch to whichever contained
  effect(s) actually implement them, in `TypeHandle` order.
- Interning parallels `RenderState`: `return_new()` shares pointers for
  equivalent effect sets via a global comparison-ordered set,
  `make_empty()`/`_empty_state` is the canonical empty set.

## API

| Method | Notes |
|---|---|
| `static CPT(RenderEffects) make(effect, ...)` (1–4 overloads) | Build from scratch |
| `static CPT(RenderEffects) make_empty()` | Canonical empty set |
| `CPT(RenderEffects) add_effect(effect) const` | Add/replace by type |
| `CPT(RenderEffects) remove_effect(TypeHandle) const` | Drop one |
| `const RenderEffect *get_effect(TypeHandle) const` / `get_effect(size_t n) const` | Lookup by type or index |
| `int find_effect(TypeHandle) const` | Index of a type, or -1 |
| `bool is_empty() const` / `size_t get_num_effects() const` / `size()` | Set inspection |
| `const RenderEffect *operator[](size_t/TypeHandle) const` | Convenience accessors |
| `bool operator<(other) const` | Ordering for interning |
| `void output(ostream&) const` / `write(ostream&, int) const` | Debug printing |

## Usage

```cpp
CPT(RenderEffects) fx = RenderEffects::make(BillboardEffect::make_axis());
node->set_effects(fx);
```

## See also

[RenderEffect](RenderEffect.md), [RenderState](RenderState.md),
[PandaNode](PandaNode.md).
