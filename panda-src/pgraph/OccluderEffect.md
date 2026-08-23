# OccluderEffect

**Source:** `panda/src/pgraph/occluderEffect.h` (+ `.I`, `.cxx`)
**Inherits:** RenderEffect

Indicates the set of [OccluderNode](OccluderNode.md)s that are enabled
(actively culling) at this point in the graph and below — functions
similarly to a `LightAttrib` or `ClipPlaneAttrib`, but as a `RenderEffect`
instead of a `RenderAttrib`.

## Behavior notes

- **Attrib-like accumulation but effect-scoped:** unlike `ClipPlaneAttrib`
  (a `RenderAttrib`, which composes additively/subtractively down the
  state stack the normal way), an `OccluderEffect` "takes effect
  immediately when it is encountered during traversal" — it can only
  *add* occluders as the cull traversal descends, never remove them. This
  is why it's a `RenderEffect` (node-local, traversal-time behavior)
  rather than a `RenderAttrib` (composed state).
- `make()` returns a cached shared "empty"/identity instance
  (`_empty_effect`, lazily constructed once) rather than allocating fresh
  — `is_identity()` checks for zero occluders.
- `add_on_occluder()`/`remove_on_occluder()` copy-construct a new
  `OccluderEffect` (immutable-value pattern, same as `RenderState`/
  `RenderAttrib`) and assert the passed `NodePath` is empty-checked and
  actually wraps an `OccluderNode`.
- Occluders are stored in an `ov_set<NodePath>` (`ordered_vector`,
  auto-sorted) so comparison/output order is deterministic.
- `require_fully_complete()` returns `true` — Bam loading must fully
  resolve all occluder `NodePath` pointers before this object is usable
  (guards against circular references during deserialization).
- `complete_pointers()` reconciles loaded occluder pointers/paths against
  the global `AttribNodeRegistry`, with a legacy code path for Bam files
  older than minor version 40 that stored plain node pointers instead of
  full `NodePath`s.

## API

| Method | Notes |
|---|---|
| `static CPT(RenderEffect) make()` | Identity/empty effect (adds nothing) |
| `int get_num_on_occluders() const` | |
| `NodePath get_on_occluder(int n) const` | Sorted by render order |
| `get_on_occluders()` | `MAKE_SEQ` iterator sequence |
| `bool has_on_occluder(const NodePath &occluder) const` | |
| `bool is_identity() const` | True if no occluders enabled |
| `CPT(RenderEffect) add_on_occluder(const NodePath &occluder) const` | Must be an `OccluderNode`; returns a new effect |
| `CPT(RenderEffect) remove_on_occluder(const NodePath &occluder) const` | Returns a new effect |

## Usage

```cpp
NodePath occluder_np = render.attach_new_node(new OccluderNode("occ"));
// ... build occluder polygon on occluder_np ...

// Enable this occluder for everything under some_node and below.
some_node.set_effect(OccluderEffect::make()->add_on_occluder(occluder_np));
```

## See also

- [RenderEffect](RenderEffect.md) — base class
- [OccluderNode](OccluderNode.md) — the node type an occluder `NodePath` must wrap
- [../pgraph/README.md](README.md) — cull pipeline overview (portal/occluder culling)
