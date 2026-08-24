# char — Panda3D's Animated Character Node

**Source:** `panda/src/char/` · Library: `libp3char` · Notify category: `char`

The renderable, scene-graph-attachable character built on top of
[chan](../chan/README.md)'s generic part/animation machinery.
[chan](../chan/README.md) defines *how* a joint hierarchy is animated and
bound to anim data in the abstract; `char` supplies the concrete
`PandaNode` subclass (`Character`) that owns that hierarchy, plus the glue
that connects joint transforms to actual mesh vertices — either
hard-skinned (one joint per vertex, via `JointVertexTransform` +
`TransformTable`/`TransformBlendTable`) or soft-skinned (many joints
blended per vertex) and morph sliders (via `CharacterSlider` +
`CharacterVertexSlider` + `SliderTable`).

## Class map

```
PandaNode (../chan/PartBundleNode.md)
└── Character                     (Character.md)

PartBundle (../chan)
└── CharacterJointBundle           (CharacterJointBundle.md) — every joint/slider in one Character

MovingPartMatrix (../chan)
└── CharacterJoint                 (CharacterJoint.md)        — one animating joint transform

MovingPartScalar (../chan)
└── CharacterSlider                (CharacterSlider.md)       — one morph slider value

VertexSlider (../gobj)
└── CharacterVertexSlider          (CharacterVertexSlider.md) — reads a CharacterSlider's value for morph target blending

VertexTransform (../gobj)
└── JointVertexTransform           (JointVertexTransform.md)  — reads a CharacterJoint's skinning matrix for soft-skinning

RenderEffect (../pgraph)
└── CharacterJointEffect           (CharacterJointEffect.md)  — auto-attached lazy-update binding between a node and its Character
```

Not documented as standalone files here: **`config_char.h/.cxx`** (module
config, summarized below) and **`p3char_composite1.cxx`,
`p3char_composite2.cxx`** (build aggregators only).

## Core concepts

**`Character` owns exactly the joint hierarchy chan describes, plus the
scene-graph and vertex-update responsibilities chan doesn't have.** Every
`Character` builds its own [CharacterJointBundle](CharacterJointBundle.md)
(a [PartBundle](../chan/PartBundle.md)) in its constructor; the
`char`-specific pieces are [CharacterJoint](CharacterJoint.md)/
[CharacterSlider](CharacterSlider.md) (which extend chan's generic
`MovingPartMatrix`/`MovingPartScalar` with net-transform composition and
skinning-matrix computation) and the update/culling logic in `Character`
itself.

**Two independent ways a joint reaches a mesh vertex.** Skinning
([JointVertexTransform](JointVertexTransform.md)) reads a joint's
`_skinning_matrix` — its transform relative to its own bind pose — and is
what actually moves rendered geometry. Socketing
(`CharacterJoint::add_net_transform()`/`add_local_transform()`) instead
copies a joint's transform onto an arbitrary `PandaNode` elsewhere in the
scene graph (e.g. attaching a prop to a hand), and is unrelated to mesh
deformation. Both can be driven by the same joint simultaneously and don't
interact.

**Lazy update via `CharacterJointEffect`.** A socketed node's transform can
go stale between the last cull-triggered `Character::update()` and an
application query. [CharacterJointEffect](CharacterJointEffect.md), silently
attached by `add_net_transform()`/`add_local_transform()`, forces an update
on read (`cull_callback()`/`adjust_transform()`) so the query always sees
the current pose without the caller needing to remember to call `update()`
first.

**Update normally happens during cull, once per frame, only if visible.**
`Character::cull_callback()` (installed via `set_cull_callback()` in the
constructor) calls `update()` for every character that survives the
view-frustum test — animating off-screen characters is skipped unless
something else (e.g. a `CharacterJointEffect` query) forces it. LOD
throttling (`set_lod_animation()`) further reduces update frequency with
camera distance by pushing a computed delay down into the underlying
`PartBundle` (`PartBundle::set_update_delay()`, see
[../chan/PartBundle.md](../chan/PartBundle.md)).

## Config variables (`config_char.h`/`.cxx`)

| Variable | Default | Purpose |
|---|---|---|
| `even-animation` (Bool) | `false` | Forces every character's vertices to recompute every frame regardless of need, trading average performance for a more uniform frame time (a profiling/debugging knob). |

## File index

| File | Documents |
|---|---|
| `character.h/.I/.cxx` | [Character](Character.md) |
| `characterJointBundle.h/.I/.cxx` | [CharacterJointBundle](CharacterJointBundle.md) |
| `characterJoint.h/.I/.cxx` | [CharacterJoint](CharacterJoint.md) |
| `characterSlider.h/.cxx` | [CharacterSlider](CharacterSlider.md) |
| `characterVertexSlider.h/.I/.cxx` | [CharacterVertexSlider](CharacterVertexSlider.md) |
| `jointVertexTransform.h/.I/.cxx` | [JointVertexTransform](JointVertexTransform.md) |
| `characterJointEffect.h/.I/.cxx` | [CharacterJointEffect](CharacterJointEffect.md) |
| `config_char.h/.cxx` | Config var, folded into this README above |
| `p3char_composite1.cxx`, `p3char_composite2.cxx` | Build aggregators only; not documented |

## Status

char — done (2026-08-23). See [../../README.md](../../README.md) for the
overall index across `panda/src/*` modules.

## See also

- [../chan/README.md](../chan/README.md) — `PartBundle`, `MovingPartMatrix`/
  `MovingPartScalar`, `AnimControl`, and everything binding-and-playback
  related that `char` builds on top of.
- [../gobj/README.md](../gobj/README.md) — `VertexTransform`, `VertexSlider`,
  `TransformTable`, `TransformBlendTable`, `SliderTable` (the geometry-side
  vertex-animation machinery `JointVertexTransform`/`CharacterVertexSlider`
  plug into).
- [../pgraph/README.md](../pgraph/README.md) — `PandaNode`, `RenderEffect`.
