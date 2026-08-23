# TexGenAttrib

**Source:** `panda/src/pgraph/texGenAttrib.h` (+ `.I`, `.cxx`)
**Inherits:** RenderAttrib

Automatically computes texture coordinates per `TextureStage`, from vertex
position/normal instead of the Geom's stored UVs — used for reflection/
refraction maps, projective texturing, point sprites, and similar effects.
`TextureStage` and `Texture` are defined in `panda/src/gobj` (undocumented).

## Behavior notes

- `Mode` is `RenderAttrib::TexGenMode` (defined in the base class, not
  here — "a hack to avoid a problem with circular includes," per the
  source comment): `M_off`, `M_eye_sphere_map`, `M_world_cube_map`,
  `M_eye_cube_map`, `M_world_normal`, `M_eye_normal`, `M_world_position`,
  `M_eye_position`, `M_point_sprite`, `M_constant` (plus deprecated/removed
  `M_unused`/`M_light_vector`).
- Holds a per-stage map (`_stages`: `TextureStage* → ModeDef`), so one
  attrib can drive different tex-gen behavior on different texture stages
  simultaneously (e.g. stage 0 uses the Geom's real UVs while stage 1 gets
  a cube-map reflection).
- `add_stage(stage, mode)` asserts `mode != M_constant` (use the
  3-argument overload for that); the 3-arg `add_stage(stage, mode,
  constant_value)` asserts `mode == M_constant`.
- `make()` memoizes a single shared empty instance, same as several other
  attribs; `make(stage, mode)` is just `make()->add_stage(stage, mode)`.
- Tracks `_num_point_sprites`/`_point_geom_rendering`/`_geom_rendering` as
  running counters so `get_geom_rendering()` can cheaply report which
  `Geom::GeomRendering` bits (e.g. `GR_point_sprite`) the attrib requires
  without re-scanning all stages every call.
- `_no_texcoords` is a redundant derived set (texture stages that don't
  need real UV data from the Geom because they're fully generated) —
  kept as a rendering-time optimization, not part of the logical state.

## API

| Signature | Notes |
|---|---|
| `static CPT(RenderAttrib) make()` | Empty — no stages generated |
| `static CPT(RenderAttrib) make(TextureStage *stage, Mode mode)` | |
| `static CPT(RenderAttrib) make_default()` | Empty |
| `CPT(RenderAttrib) add_stage(TextureStage *stage, Mode mode) const` | `mode != M_constant` |
| `CPT(RenderAttrib) add_stage(TextureStage *stage, Mode mode, const LTexCoord3 &constant_value) const` | `mode == M_constant` only |
| `CPT(RenderAttrib) remove_stage(TextureStage *stage) const` | |
| `bool is_empty() const` / `has_stage(TextureStage*) const` | |
| `Mode get_mode(TextureStage *stage) const` | `M_off` if stage not present |
| `bool has_gen_texcoord_stage(TextureStage*) const` | |
| `const LTexCoord3 &get_constant_value(TextureStage*) const` | |
| `int get_geom_rendering(int geom_rendering) const` | OR's in required `Geom::GeomRendering` bits |

## Usage

```cpp
PT(TextureStage) stage = TextureStage::get_default();
node_path.set_tex_gen(stage, TexGenAttrib::M_world_cube_map); // NodePath wrapper
// or directly:
node_path.set_attrib(TexGenAttrib::make(stage, RenderAttrib::M_eye_sphere_map));
```

## See also

[README — the state pipeline](README.md#the-state-pipeline),
[RenderAttrib](RenderAttrib.md), [TexMatrixAttrib](TexMatrixAttrib.md),
[TextureAttrib](TextureAttrib.md)
