# chan — Panda3D's Animation Channel & Binding System

**Source:** `panda/src/chan/` · Library: `libp3chan` · Notify category: `chan`

Defines how animation data is stored, played back, and connected to a
skeleton. Two parallel hierarchies meet here: an **anim** side (`AnimGroup` →
`AnimBundle`/`AnimChannel*`, describing *what the animation data is*, loaded
from an egg/bam file or built procedurally) and a **part** side (`PartGroup`
→ `PartBundle`/`MovingPart*`, describing *what can be animated* — a
skeleton's joints and morph sliders). [AnimControl](AnimControl.md) is the
runtime object created when the two sides are bound together via
`PartBundle::bind_anim()`. This module has no scene-graph rendering
responsibility of its own; [char](../char/README.md) builds the actual
renderable `Character` node on top of it.

## Class map

```
TypedWritableReferenceCount, Namable
└── AnimGroup                          (AnimGroup.md)
    ├── AnimBundle                     (AnimBundle.md)          — root of one anim hierarchy
    └── AnimChannelBase                (AnimChannelBase.md)
        └── AnimChannel<SwitchType>                             — template, no standalone page
            ├── AnimChannelMatrix      (AnimChannelMatrix.md)   — = AnimChannel<ACMatrixSwitchType>
            │   ├── AnimChannelMatrixDynamic     (AnimChannelMatrixDynamic.md)
            │   ├── AnimChannelMatrixFixed       (AnimChannelMatrixFixed.md)
            │   ├── AnimChannelMatrixXfmTable    (AnimChannelMatrixXfmTable.md)
            │   └── AnimChannelFixed<ACMatrixSwitchType>  (AnimChannelFixed.md)
            └── AnimChannelScalar      (AnimChannelScalar.md)   — = AnimChannel<ACScalarSwitchType>
                ├── AnimChannelScalarDynamic     (AnimChannelScalarDynamic.md)
                ├── AnimChannelScalarTable       (AnimChannelScalarTable.md)
                └── AnimChannelFixed<ACScalarSwitchType>  (AnimChannelFixed.md)

PandaNode (../pgraph)
└── AnimBundleNode                     (AnimBundleNode.md)      — wraps an AnimBundle in the scene graph

TypedWritableReferenceCount, Namable
└── PartGroup                          (PartGroup.md)
    ├── PartBundle                     (PartBundle.md)          — root of one animatable object; see ../char/CharacterJointBundle.md
    └── MovingPartBase                 (MovingPartBase.md)
        └── MovingPart<SwitchType>                               — template, no standalone page
            ├── MovingPartMatrix       (MovingPartMatrix.md)    — = MovingPart<ACMatrixSwitchType>; see ../char/CharacterJoint.md
            └── MovingPartScalar       (MovingPartScalar.md)    — = MovingPart<ACScalarSwitchType>; see ../char/CharacterSlider.md

PandaNode (../pgraph)
└── PartBundleNode                     (PartBundleNode.md)      — base of ../char/Character.md

TypedReferenceCount, AnimInterface, Namable
└── AnimControl                        (AnimControl.md)         — one part↔anim binding's playback state

ReferenceCount
└── PartBundleHandle                   (PartBundleHandle.md)    — indirection so flatten can't invalidate an Actor's PartBundle*

CopyOnWriteObject
└── AnimPreloadTable                   (AnimPreloadTable.md)    — offline-built metadata enabling async binds

ModelLoadRequest (../pgraph)
└── BindAnimRequest                    (BindAnimRequest.md)     — the async load-and-bind task itself

(value types, no inheritance)
├── AnimControlCollection              (AnimControlCollection.md) — named AnimControl lookup
└── PartSubset                         (PartSubset.md)          — restricts a bind_anim() to named joints

Free function: auto_bind()             (auto_bind.md)           — scene-graph-wide anim/part auto-matching
```

**Templates fold into their concrete instantiation's page** (no standalone
doc): `AnimChannel<SwitchType>` is documented in full on
[AnimChannelMatrix.md](AnimChannelMatrix.md) (the `ACMatrixSwitchType`
instantiation); [AnimChannelScalar.md](AnimChannelScalar.md) covers only
what differs for `ACScalarSwitchType`. Likewise `MovingPart<SwitchType>` is
documented in full on [MovingPartMatrix.md](MovingPartMatrix.md), with
[MovingPartScalar.md](MovingPartScalar.md) covering only the scalar-specific
differences. `AnimChannelFixed<SwitchType>` is documented once on its own
page covering both instantiations.

Not documented as standalone files here (out of scope / internal-only):
- **`config_chan.h/.cxx`** — module config; variables summarized below.
- **`vector_PartGroupStar.h/.cxx`** — not a class, just a `pvector<PartGroup*>`
  export-shim typedef. Mentioned in [PartGroup.md](PartGroup.md).
- **`p3chan_composite1.cxx`, `p3chan_composite2.cxx`** — build-system
  aggregator files that `#include` the module's other `.cxx` files; not
  independent translation units worth documenting.

## Core concepts

**Anim data and part hierarchies are structurally parallel, matched by
name.** An `AnimBundle`'s tree of `AnimGroup`/`AnimChannel` nodes and a
`PartBundle`'s tree of `PartGroup`/`MovingPart` nodes are expected to mirror
each other joint-for-joint, matched at bind time by name at each level (see
`PartGroup::check_hierarchy()`, documented in [PartGroup.md](PartGroup.md)).
A mismatch (missing joint, wrong type — matrix channel vs. scalar joint)
fails the bind for that branch, controlled by `hierarchy_match_flags`
(`PartGroup::HMF_*` constants).

**Binding creates an `AnimControl`; nothing plays until playback is explicitly started.**
`PartBundle::bind_anim()` (or the async `load_bind_anim()`) is the one path
that produces an [AnimControl](AnimControl.md). The control's own
`play()`/`loop()`/`pose()` (inherited from `AnimInterface`) then drive
playback; blend weight (`PartBundle::set_control_effect()`) determines how
much a bound-but-not-"current" animation actually affects the final pose
when multiple controls are active (`set_anim_blend_flag(true)`).

**Async binding needs a preload table.** `PartBundle::load_bind_anim()` can
only actually go async if the target animation's basename is registered in
an [AnimPreloadTable](AnimPreloadTable.md) — that's where the frame
count/rate needed to make an immediately-usable pending `AnimControl` come
from, before the real file has loaded. Without a table hit, it silently
falls back to synchronous loading. [BindAnimRequest](BindAnimRequest.md) is
the `AsyncTask` that actually performs the deferred load-and-bind.

**Two switch types, one set of templates.** `ACMatrixSwitchType` and
`ACScalarSwitchType` are compile-time policy tags (no runtime instances)
that specialize `AnimChannel<SwitchType>` and `MovingPart<SwitchType>` for
`LMatrix4` (joint transforms) vs. `PN_stdfloat` (morph slider values)
respectively — see [AnimChannelMatrix.md](AnimChannelMatrix.md) for the
mechanism in detail.

**`auto_bind()` is the shortcut for the common case.** Rather than manually
walking a loaded scene graph to find matching `AnimBundle`/`PartBundle`
pairs, [auto_bind()](auto_bind.md) does this automatically, matched by bundle name,
and fills an [AnimControlCollection](AnimControlCollection.md) with the
results.

## Config variables (`config_chan.h`/`.cxx`)

| Variable | Default | Purpose |
|---|---|---|
| `compress-channels` (Bool) | `false` | Lossy-compress animation channels when writing a bam file (file size only, not runtime memory). |
| `compress-chan-quality` (Int) | `95` | Quality 0–100 for the above; values above 95 rarely help. Special debug-only values above 100 bypass FFT compression entirely. |
| `read-compressed-channels` (Bool) | `true` | Set `false` to skip decompression support entirely, trading animation fidelity for load speed. |
| `interpolate-frames` (Bool) | `false` | Blend between frames instead of holding each one; overridable per-bundle via `PartBundle::set_frame_blend_flag()`. |
| `restore-initial-pose` (Bool) | `true` | Whether zeroing all control effects returns to the default pose or freezes at the last-computed one. |
| `async-bind-priority` (Int) | `100` | Task priority for `BindAnimRequest`s relative to other async loads (texture/model); higher loads sooner. |

## File index

| File | Documents |
|---|---|
| `animGroup.h/.I/.cxx` | [AnimGroup](AnimGroup.md) |
| `animBundle.h/.I/.cxx` | [AnimBundle](AnimBundle.md) |
| `animBundleNode.h/.I/.cxx` | [AnimBundleNode](AnimBundleNode.md) |
| `animChannelBase.h/.I/.cxx` | [AnimChannelBase](AnimChannelBase.md) |
| `animChannel.h/.I/.cxx` | [AnimChannelMatrix](AnimChannelMatrix.md), [AnimChannelScalar](AnimChannelScalar.md) (template instantiations) |
| `animChannelFixed.h/.I/.cxx` | [AnimChannelFixed](AnimChannelFixed.md) |
| `animChannelMatrixDynamic.h/.I/.cxx` | [AnimChannelMatrixDynamic](AnimChannelMatrixDynamic.md) |
| `animChannelMatrixFixed.h/.I/.cxx` | [AnimChannelMatrixFixed](AnimChannelMatrixFixed.md) |
| `animChannelMatrixXfmTable.h/.I/.cxx` | [AnimChannelMatrixXfmTable](AnimChannelMatrixXfmTable.md) |
| `animChannelScalarDynamic.h/.I/.cxx` | [AnimChannelScalarDynamic](AnimChannelScalarDynamic.md) |
| `animChannelScalarTable.h/.I/.cxx` | [AnimChannelScalarTable](AnimChannelScalarTable.md) |
| `animControl.h/.I/.cxx` | [AnimControl](AnimControl.md) |
| `animControlCollection.h/.I/.cxx` | [AnimControlCollection](AnimControlCollection.md) |
| `animPreloadTable.h/.I/.cxx` | [AnimPreloadTable](AnimPreloadTable.md) |
| `auto_bind.h/.cxx` | [auto_bind](auto_bind.md) |
| `bindAnimRequest.h/.I/.cxx` | [BindAnimRequest](BindAnimRequest.md) |
| `movingPartBase.h/.I/.cxx` | [MovingPartBase](MovingPartBase.md) |
| `movingPart.h/.I` | [MovingPartMatrix](MovingPartMatrix.md), [MovingPartScalar](MovingPartScalar.md) (template instantiations) |
| `movingPartMatrix.h/.I/.cxx` | [MovingPartMatrix](MovingPartMatrix.md) |
| `movingPartScalar.h/.I/.cxx` | [MovingPartScalar](MovingPartScalar.md) |
| `partGroup.h/.I/.cxx` | [PartGroup](PartGroup.md) |
| `partBundle.h/.I/.cxx` | [PartBundle](PartBundle.md) |
| `partBundleHandle.h/.I/.cxx` | [PartBundleHandle](PartBundleHandle.md) |
| `partBundleNode.h/.I/.cxx` | [PartBundleNode](PartBundleNode.md) |
| `partSubset.h/.I/.cxx` | [PartSubset](PartSubset.md) |
| `vector_PartGroupStar.h/.cxx` | Typedef shim, folded into [PartGroup.md](PartGroup.md) |
| `config_chan.h/.cxx` | Config vars, folded into this README above |
| `p3chan_composite1.cxx`, `p3chan_composite2.cxx` | Build aggregators only; not documented |

## Status

chan — done (2026-08-23). See [../../README.md](../../README.md) for the
overall index across `panda/src/*` modules.

## See also

- [../char/README.md](../char/README.md) — `Character`, the renderable node
  built on top of `PartBundle`/`MovingPart`.
- [../pgraph/README.md](../pgraph/README.md) — `PandaNode`, `TransformState`,
  and `ModelLoadRequest` (`BindAnimRequest`'s base class).
- [../event/README.md](../event/README.md) — `AsyncTask`/`AsyncTaskManager`,
  what a `BindAnimRequest` runs on.
