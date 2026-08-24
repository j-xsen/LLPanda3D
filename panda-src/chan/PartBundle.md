# PartBundle

**Source:** `panda/src/chan/partBundle.h` / `.I` / `.cxx`
**Inherits:** [PartGroup](PartGroup.md)
**Inherited by:** `../char/CharacterJointBundle.md`

The root of a `MovingPart` hierarchy — defines the full set of joints/
sliders that make up one animatable object (a character's skeleton). This is
the central class of the `chan` module: it owns the per-frame blend state
across pipeline stages, drives `update()`, and is the entry point for
binding animations.

## Behavior notes

- **`bind_anim()` requires an exact name and hierarchy match by default.**
  `do_bind_anim()` checks the bundle's name against the anim's root name
  (skippable via `HMF_ok_wrong_root_name`), then calls
  [PartGroup::check_hierarchy()](PartGroup.md) — see that page's notes on
  alphabetical-order matching and the `"morph"` special case. On success, it
  picks a free channel index (`pick_channel_index()`), builds a `BitArray`
  of which joints actually got bound (relevant for `PartSubset`-restricted
  binds), and creates/wires up the returned `AnimControl` via
  `AnimControl::setup_anim()`.
- **`load_bind_anim()`'s async path depends entirely on the preload table.**
  It only attempts an asynchronous bind if `allow_async` is true, threading
  is supported, *and* the requested animation's basename is found in
  `get_anim_preload()` (see [AnimPreloadTable](AnimPreloadTable.md)) — that
  table is what supplies the frame rate/count needed to construct a usable
  `AnimControl` before the file has actually loaded. Without a preload-table
  hit, it silently falls back to a synchronous `Loader::load_sync()` +
  `bind_anim()`, even if `allow_async` was requested. The async path
  dispatches a [BindAnimRequest](BindAnimRequest.md) to the given `Loader`.
- **`apply_transform()` caches its result per `TransformState` pointer** in
  `_applied_transforms` (a `pmap` keyed by `owner_less` on a weak pointer) —
  calling it twice with the same `TransformState*` returns the same
  duplicated-and-transformed `PartBundle`, rather than creating a new copy
  each time. This is what lets many `Actor` instances loaded with different
  transforms share `PartBundle` results when the transform matches.
  Requesting the identity transform returns `this` unchanged (no copy at
  all).
- **`set_anim_blend_flag(false)` (the default) forcibly narrows to one active
  animation.** Turning the flag off, when more than one `AnimControl`
  currently has nonzero effect, keeps only `_last_control_set` and stops
  every other control that overlaps its bound joints
  (`clear_and_stop_intersecting()` — an overlap is any control whose
  `bound_joints` `BitArray` shares set bits with the target's). Starting a
  new animation while the flag is `false` implicitly sets that animation's
  control effect to 1.0 via `control_activated()`, with no explicit
  `set_control_effect()` call needed.
- **Zero-effect controls aren't removed from tracking, just excluded from
  blending.** `set_control_effect(control, 0.0f)` erases the entry from
  `CData::_blend`, but the `AnimControl` itself keeps running (consuming no
  extra CPU beyond its own timer) until explicitly stopped — see the
  `clear_control_effects()` doc comment: controls "may still be in the
  'playing' state" and resume exactly where they left off if reassociated.
- **`update()` is throttled by `_update_delay`** (set via
  `set_update_delay()`, normally only from `Character::set_lod_animation()`
  in the `char` module) — it skips recomputation if less time than the delay
  has passed since `_last_update`, *unless* `_anim_changed` is set (e.g. a
  new bind or `xform()` happened). `force_update()` bypasses both checks
  unconditionally.
- **`freeze_joint()`/`control_joint()`/`release_joint()` look up the joint
  by name via `find_child()`** (recursive) and delegate to
  `PartGroup::apply_freeze()`/`apply_freeze_matrix()`/`apply_freeze_scalar()`/
  `apply_control()`/`clear_forced_channel()` — they return `false` if no
  child with that name exists, or if the child's type doesn't support the
  requested operation (a scalar-typed part can't `apply_freeze_matrix()`,
  for instance).
- **CData is `PipelineCycler`-backed** (staged across the render pipeline),
  holding `_blend_type`, `_anim_blend_flag`, `_frame_blend_flag`,
  `_root_xform`, the `ChannelBlend` map (`AnimControl* → PN_stdfloat`
  weight), `_net_blend` (sum of all weights, kept current by
  `recompute_net_blend()`), and `_anim_changed`/`_last_update` bookkeeping.
  `_last_control_set` and the `_blend` map are explicitly *not* copied by
  `PartBundle`'s own copy constructor — the comment notes the `CData` copy
  constructor "is not used by the PartBundle copy constructor," only static
  config fields (`_blend_type`, both blend flags, `_root_xform`) carry over
  to a duplicate.
- **`anim-blend-type` config variable sets the default `BlendType`**
  (`BT_normalized_linear`), read once into each fresh `CData`.

## API

### Binding animations
| Signature | Notes |
|---|---|
| `PT(AnimControl) bind_anim(AnimBundle *anim, int hierarchy_match_flags = 0, const PartSubset &subset = PartSubset())` | Synchronous bind; returns `nullptr` on hierarchy mismatch |
| `PT(AnimControl) load_bind_anim(Loader *loader, const Filename &filename, int hierarchy_match_flags, const PartSubset &subset, bool allow_async)` | Load-from-disk + bind, optionally async; see Behavior notes |
| `void wait_pending()` | Blocks until every nonzero-effect `AnimControl` finishes any pending async bind |
| `CPT(AnimPreloadTable) get_anim_preload() const` / `PT(AnimPreloadTable) modify_anim_preload()` | Table enabling async binds; see [AnimPreloadTable](AnimPreloadTable.md) |
| `void set_anim_preload(AnimPreloadTable *table)` / `void clear_anim_preload()` | |
| `void merge_anim_preloads(const PartBundle *other)` | Copy-on-write merge of another bundle's preload entries |

### Blend configuration
| Signature | Notes |
|---|---|
| `void set_blend_type(BlendType bt)` / `BlendType get_blend_type() const` | See `BlendType` enum below |
| `void set_anim_blend_flag(bool)` / `bool get_anim_blend_flag() const` | Allow multiple simultaneous animations; default `false` |
| `void set_frame_blend_flag(bool)` / `bool get_frame_blend_flag() const` | Interpolate between frames; default from `interpolate-frames` config var |
| `void set_control_effect(AnimControl *control, PN_stdfloat effect)` / `PN_stdfloat get_control_effect(AnimControl *control) const` | Per-control blend weight |
| `void clear_control_effects()` | Zeroes all control effects (doesn't stop the controls) |

### Enum `BlendType`
| Value | Meaning |
|---|---|
| `BT_linear` | Componentwise matrix average; can stretch/squash |
| `BT_normalized_linear` | Blends pos/rot without scale/shear, reapplies those separately (default) |
| `BT_componentwise` | Decomposes to scale/hpr/pos/shear, blends each independently |
| `BT_componentwise_quat` | Like componentwise, but rotation blends as a quaternion |

### Transform / joints
| Signature | Notes |
|---|---|
| `void set_root_xform(const LMatrix4&)` / `void xform(const LMatrix4&)` / `const LMatrix4 &get_root_xform() const` | `xform()` composes onto the existing root transform and recurses `do_xform()` into every part |
| `PT(PartBundle) apply_transform(const TransformState *transform)` | Returns a cached, transformed duplicate bundle; see Behavior notes |
| `bool freeze_joint(const std::string &joint_name, const TransformState *transform)` | Also overloaded for `(pos, hpr, scale)` and `(PN_stdfloat value)` |
| `bool control_joint(const std::string &joint_name, PandaNode *node)` | Joint follows the node's transform |
| `bool release_joint(const std::string &joint_name)` | Undoes freeze/control |

### Per-frame update
| Signature | Notes |
|---|---|
| `bool update()` | Throttled by `_update_delay`; returns whether anything changed |
| `bool force_update()` | Unconditional; also used internally after `apply_transform()` and after Bam load (`finalize()`) |

### Nodes
| Signature | Notes |
|---|---|
| `int get_num_nodes() const` / `PartBundleNode *get_node(int n) const` | Every [PartBundleNode](PartBundleNode.md) sharing this bundle |

## Usage

```cpp
PartBundle *bundle = new PartBundle("skeleton");
MovingPartMatrix *hips = new MovingPartMatrix(bundle, "hips", LMatrix4::ident_mat());
bundle->sort_descendants();

AnimBundle *anim = new AnimBundle("skeleton", 24.0f, 30);
PT(AnimControl) control = bundle->bind_anim(anim);
if (control != nullptr) {
  control->loop(true);
  bundle->update();  // call once per frame
}
```

## See also

[PartGroup](PartGroup.md), [MovingPartMatrix](MovingPartMatrix.md),
[MovingPartScalar](MovingPartScalar.md), [PartBundleNode](PartBundleNode.md),
[PartBundleHandle](PartBundleHandle.md), [PartSubset](PartSubset.md),
[AnimControl](AnimControl.md), [AnimPreloadTable](AnimPreloadTable.md),
[BindAnimRequest](BindAnimRequest.md), [AnimBundle](AnimBundle.md),
[../char/CharacterJointBundle.md](../char/CharacterJointBundle.md),
[README.md](README.md)
