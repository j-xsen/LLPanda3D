# TexProjectorEffect

**Source:** `panda/src/pgraph/texProjectorEffect.h` (+ `.I`, `.cxx`)
**Inherits:** RenderEffect

Automatically applies a computed texture matrix to a specified
`TextureStage`, each frame, based on the relative transform between two
nodes ("from" and "to"). If the "to" node is a `LensNode`
(e.g. [Camera](Camera.md)), its lens projection matrix is folded in too —
enabling projective texturing effects like a flashlight cone or an
image-based shadow projected onto walls. Source: `panda/src/mathutil` for
`Lens` itself (undocumented module).

## Behavior notes

- Holds a map from `TextureStage*` to a `(from, to, lens_index)`
  definition; `add_stage()`/`remove_stage()` follow the standard
  immutable-value pattern (copy-construct + modify + intern via
  `return_new()`).
- `make()` with no stages returns a cached shared "empty" instance
  (`_empty_effect`), same optimization pattern as `OccluderEffect` —
  `is_empty()` checks for zero stages.
- At cull time, each stage's `from`→`to` relative transform (composed with
  the "to" node's lens projection matrix if it's a `LensNode`) is computed
  and applied as a `TexMatrixAttrib` for that `TextureStage`.
- **Typical simple use:** a standalone `PandaNode` (not necessarily in the
  visible graph) that is animated (e.g. via a `LerpInterval`) purely to
  drive an object's texture-coordinate transform, avoiding an explicit
  per-frame `NodePath::set_tex_transform()` call.
- **Advanced use, per the header doc:** combine with a `TexGenAttrib` set
  to `M_world_position` (copies each vertex's world position into its
  texture coordinates) so the projector can convert those world
  coordinates into the "to" node's relative space — makes a texture appear
  to "follow" a node through the world, or (with a `LensNode` as "to")
  project a texture through a virtual camera onto the scene, e.g. a
  flashlight or projected shadow.
- If "to" is a `LensNode`, `lens_index` selects which of its (possibly
  multiple) `Lens` objects supplies the projection matrix.

## API

| Method | Notes |
|---|---|
| `static CPT(RenderEffect) make()` | Empty effect (no stages) |
| `CPT(RenderEffect) add_stage(TextureStage *stage, const NodePath &from, const NodePath &to, int lens_index = 0) const` | Replaces existing definition for `stage` if present; returns new effect |
| `CPT(RenderEffect) remove_stage(TextureStage *stage) const` | Returns new effect |
| `bool is_empty() const` | |
| `bool has_stage(TextureStage *stage) const` | |
| `NodePath get_from(TextureStage *stage) const` | |
| `NodePath get_to(TextureStage *stage) const` | |
| `int get_lens_index(TextureStage *stage) const` | |

## Usage

```cpp
// Project a spotlight's-eye view onto the scene through TextureStage ts,
// using camera_np (a LensNode) as the projector.
model_np.set_effect(
  TexProjectorEffect::make()->add_stage(ts, NodePath(), camera_np));
model_np.set_tex_gen(ts, TexGenAttrib::M_world_position);
```

## See also

- [RenderEffect](RenderEffect.md) — base class
- [Camera](Camera.md), [LensNode](LensNode.md) — common "to" node for projective effects
- [../pgraph/README.md](README.md) — module overview
