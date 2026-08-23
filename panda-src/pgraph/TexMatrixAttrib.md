# TexMatrixAttrib

**Source:** `panda/src/pgraph/texMatrixAttrib.h` (+ `.I`, `.cxx`)
**Inherits:** RenderAttrib

Applies a transform matrix to UV coordinates before rendering, per
`TextureStage` — used to scroll/rotate/scale a texture independently of
the underlying geometry (e.g. animated water, scrolling UI textures).

## Behavior notes

- Per-stage entries (`StageNode`: `TextureStage*` + `TransformState*` +
  `override`) are kept in an `ov_set` sorted by stage pointer
  (`CompareTextureStagePointer`) — same "sorted vector, not a map" pattern
  used elsewhere in `pgraph` for cheap composition.
- `get_override(stage)` raises an assertion (`nassert_raise`) if the stage
  isn't present, rather than returning a sentinel — check `has_stage()`
  first.
- The legacy `make(const LMatrix4 &mat)` and no-stage `get_mat()` apply to
  an implicit "default" stage; the modern per-stage interface is
  `add_stage(stage, transform, override)`/`get_mat(stage)`/
  `get_transform(stage)`.
- `get_geom_rendering()` only adds a bit
  (`Geom::GR_point_sprite_tex_matrix`) when the Geom already has point
  sprites and this attrib is non-empty — point sprites need special UV
  handling for a tex matrix to apply correctly to their auto-generated
  coordinates.

## API

| Signature | Notes |
|---|---|
| `static CPT(RenderAttrib) make()` | Empty (identity for all stages) |
| `static CPT(RenderAttrib) make(const LMatrix4 &mat)` | Legacy single-stage form |
| `static CPT(RenderAttrib) make(TextureStage *stage, const TransformState *transform)` | |
| `static CPT(RenderAttrib) make_default()` | Empty |
| `CPT(RenderAttrib) add_stage(TextureStage *stage, const TransformState *transform, int override=0) const` | |
| `CPT(RenderAttrib) remove_stage(TextureStage *stage) const` | |
| `bool is_empty() const` / `has_stage(TextureStage*) const` | |
| `int get_num_stages() const` / `TextureStage *get_stage(int n) const` / `MAKE_SEQ(get_stages, ...)` | |
| `const LMatrix4 &get_mat() const` / `get_mat(TextureStage*) const` | |
| `CPT(TransformState) get_transform(TextureStage*) const` | |
| `int get_override(TextureStage *stage) const` | Asserts if stage not present |
| `int get_geom_rendering(int geom_rendering) const` | |

## Usage

```cpp
PT(TextureStage) stage = TextureStage::get_default();
node_path.set_tex_transform(stage, TransformState::make_pos(LVector3(0.1, 0, 0))); // NodePath wrapper, scrolls U
```

## See also

[README — the state pipeline](README.md#the-state-pipeline),
[RenderAttrib](RenderAttrib.md), [TexGenAttrib](TexGenAttrib.md),
[TextureAttrib](TextureAttrib.md)
