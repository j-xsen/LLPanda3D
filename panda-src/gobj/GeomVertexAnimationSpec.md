# GeomVertexAnimationSpec

**Source:** `panda/src/gobj/geomVertexAnimationSpec.h` (+ `.I`, `.cxx`)
**Inherits:** [GeomEnums](README.md)

A small value-type carried by [GeomVertexFormat](GeomVertexFormat.md)
describing how — if at all — a `GeomVertexData` using that format encodes
vertex animation (soft-skinned skeleton animation and morphs/blend shapes).
Setting this doesn't itself perform any animation; it only declares the
data layout convention the animation system should follow when it does.

## Behavior notes

- Three mutually-exclusive states set via `set_none()`, `set_panda()`, or
  `set_hardware(int num_transforms, bool indexed_transforms)`:
  - `AT_none` — no vertex animation; the default.
  - `AT_panda` — animation is computed on the CPU by Panda (see
    [TransformBlendTable](TransformBlendTable.md)), and the results are
    written directly into the ordinary vertex columns before rendering.
  - `AT_hardware` — animation is passed down to the GPU: `num_transforms`
    is how many transform-matrix columns are supplied per vertex (the
    matrix palette size needed per vertex), and `indexed_transforms`
    indicates whether an additional index column selects which palette
    entries apply (indexed vs. a fixed per-vertex transform count).
- Supports `operator <` / `compare_to()` (used because `GeomVertexFormat`
  instances — which embed this — are globally interned/hashed; two formats
  compare equal only if their animation specs match exactly, including
  `num_transforms`/`indexed_transforms` for the hardware case).
- `write_datagram()`/`fillin()` serialize as: `uint8` animation type,
  `uint16` num_transforms, `bool` indexed_transforms — part of the `.bam`
  file format for `GeomVertexFormat`.
- Whether `AT_hardware` is actually usable at runtime depends on GSG
  capability and the `hardware-animated-vertices`/`matrix-palette` config
  vars (see [../README.md](README.md) config table) — this class just
  describes the format, it doesn't check hardware support itself.

## API

| Method | Notes |
|---|---|
| `GeomVertexAnimationSpec()` | Defaults to `AT_none`. |
| `get_animation_type()` | Returns `AT_none` / `AT_panda` / `AT_hardware`. |
| `get_num_transforms()` | Meaningful only for `AT_hardware`. |
| `get_indexed_transforms()` | Meaningful only for `AT_hardware`. |
| `set_none()` | Switch to `AT_none`. |
| `set_panda()` | Switch to `AT_panda`. |
| `set_hardware(int num_transforms, bool indexed_transforms)` | Switch to `AT_hardware` with the given palette parameters. |
| `operator <`, `operator ==`/`!=`, `compare_to()` | Total ordering/equality, used by `GeomVertexFormat` interning. |
| `output(ostream&)` | e.g. `"none"`, `"panda"`, `"hardware(4, 1)"`. |

## Usage

```cpp
GeomVertexAnimationSpec anim;
anim.set_hardware(4, true);  // 4-bone palette, indexed
// typically passed into GeomVertexFormat::make_from_animation() or similar
// format-construction helpers rather than mutated on an existing format.
```

## See also

- [GeomVertexFormat](GeomVertexFormat.md) — carries one of these per format
- [TransformBlendTable](TransformBlendTable.md), [TransformTable](TransformTable.md) — the actual CPU/hardware skinning data this spec describes the layout for
