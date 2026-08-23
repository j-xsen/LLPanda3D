# CompassEffect

**Source:** `panda/src/pgraph/compassEffect.h` (+ `.I`, `.cxx`)
**Inherits:** RenderEffect

A `RenderEffect` that makes a node inherit selected transform components
(pos/rotation/scale, individually or in combination) from a reference node
elsewhere in the graph — or the scene graph root — instead of from its
actual parent chain. In its purest form (default: rotation only, no
reference) it keeps the node's rotation fixed relative to the world root
regardless of ancestor rotations, "always pointing the same direction"
like a magnetic compass — hence the name.

## Behavior notes

- `properties` is a bitmask of the `Properties` enum: `P_x`/`P_y`/`P_z`
  (individually) or `P_pos` (all three), `P_rot`, `P_sx`/`P_sy`/`P_sz`
  (individually) or `P_scale` (all three), or `P_all`. Default is `P_rot`.
- `reference` empty means inherit from the scene graph root (identity
  transform); non-empty means inherit from that specific NodePath's net
  transform instead.
- Implemented via `adjust_transform()`/`cull_callback()`: it computes what
  the node's *net* transform ought to be (steal wanted components from the
  reference, keep the rest from the true parent chain), then back-computes
  a local transform correction so that composing through the real parent
  chain still produces that net transform. When only pos, or only rot+scale
  are stolen, there's a fast path; mixed/partial component selection
  requires decomposing both transforms into quat+scale (falls back to
  "steal everything but pos" if either transform isn't decomposable, e.g.
  a general/skewed matrix).
- `safe_to_transform()` returns `false` — same reasoning as
  `BillboardEffect`: this is a per-frame, graph-position-dependent
  correction, not something safe to bake in via flattening.
- **Warning from the header comment:** using pos or scale modes can move
  the node far from its normal bounding volume, breaking culling — may
  need an explicit large/infinite bounding volume on the effect node in
  that case.
- Rotation cannot be adjusted per-axis (`P_rot` is all-or-nothing) — doing
  so per-component "is just asking for trouble" per the source comment.

## API

| Method | Notes |
|---|---|
| `static CPT(RenderEffect) make(const NodePath &reference, int properties = P_rot)` | `reference` empty = scene graph root |
| `const NodePath &get_reference() const` | |
| `int get_properties() const` | Bitmask of `Properties` |

`Properties` enum: `P_x`, `P_y`, `P_z`, `P_pos` (=x|y|z), `P_rot`, `P_sx`,
`P_sy`, `P_sz`, `P_scale` (=sx|sy|sz), `P_all`.

## Usage

```cpp
// Keep this node's rotation fixed relative to the world, ignoring any
// rotation applied by its ancestors (the classic "compass" use case).
node_path.set_effect(CompassEffect::make(NodePath()));

// Inherit position from a specific reference node too.
node_path.set_effect(CompassEffect::make(reference_np, CompassEffect::P_rot | CompassEffect::P_pos));
```

## See also

- [RenderEffect](RenderEffect.md) — base class
- [../pgraph/README.md](README.md) — cull pipeline / state pipeline overview
