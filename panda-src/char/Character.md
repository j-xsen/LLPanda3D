# Character

**Source:** `panda/src/char/character.h` / `.I` / `.cxx`
**Inherits:** [../chan/PartBundleNode.md](../chan/PartBundleNode.md)

An animated character node, with skeleton-morph animation and either
soft-skinned or hard-skinned vertices. A `Character`'s constructor
automatically creates and owns a [CharacterJointBundle](CharacterJointBundle.md)
(a [../chan/PartBundle.md](../chan/PartBundle.md) subclass holding the joint/
slider hierarchy); geometry parented below the `Character` node references
that hierarchy's [CharacterJoint](CharacterJoint.md)/[CharacterSlider](CharacterSlider.md)
parts indirectly, through [JointVertexTransform](JointVertexTransform.md)/
[CharacterVertexSlider](CharacterVertexSlider.md) objects installed in the
geometry's `TransformTable`/`TransformBlendTable`/`SliderTable`. This is the
largest and most central class in the module — everything else here exists
to support it.

## Behavior notes

- **`update()` is deduplicated per frame.** It compares
  `ClockObject::get_global_clock()->get_frame_time()` against a stored
  `_last_auto_update` timestamp and does nothing if they match — so calling
  `update()` (or relying on the automatic `cull_callback()` update, see
  below) many times in the same frame only does the real work once.
  `force_update()` bypasses this check entirely and always recomputes.
- **Animation normally happens during the cull traversal, not render.**
  `Character` sets `set_cull_callback()` in its constructor and implements
  `cull_callback()` to call `update()` on every character that survives the
  view-frustum test that frame — an optimization to skip animating
  characters that aren't visible. A character that animates itself into view
  from off-frustum won't be caught by this.
- **LOD animation throttling** (`set_lod_animation()`/`clear_lod_animation()`):
  when active, `cull_callback()` computes the character's distance from the
  camera's LOD/cull center (or the camera itself) each frame, linearly
  interpolates a delay between `near_distance` (full rate) and
  `far_distance` (`delay_factor` seconds between updates), and pushes that
  delay down into every bundle via `PartBundle::set_update_delay()`. If
  multiple cameras view the character in one frame, the closest one wins
  (tracked via `_view_frame`/`_view_distance2`).
- **`even_animation` config variable** (see `config_char.h`, folded into the
  module README) switches `do_update()` from calling each bundle's
  `update()` (only recomputes what actually changed) to `force_update()`
  (recomputes unconditionally) — a debugging/profiling knob to make
  animation cost uniform across frames.
- **`make_copy()` vs `dupe_for_flatten()` vs `copy_subgraph()`:** `make_copy()`
  deep-copies the joint bundle(s) and builds fresh dynamic vertex arrays,
  but explicitly does **not** copy the original's geometry — the header
  warns the new `Character` "won't look like much" until you use
  `copy_subgraph()` instead, which does copy geometry (via the internal
  `r_copy_children()`/`r_copy_char()` machinery) and redirects every `Geom`'s
  `TransformTable`/`TransformBlendTable`/`SliderTable` to point at the new
  character's joints/sliders. `dupe_for_flatten()` is similar to
  `make_copy()` but shares internal data with the original rather than
  necessarily duplicating it, for use by the scene graph flattener.
- **`combine_with()` merges two `Character`s of the exact same type** by
  calling `steal_bundles()` (inherited from `PartBundleNode`) rather than
  doing a generic `PandaNode` combine — this only fires for exact-type
  matches (`is_exact_type()`), not subclasses.
- **`calc_tight_bounds()` force-updates before measuring**, then works
  around a self-inflicted cache-invalidation bug: calling `update_to_now()`
  invalidates this node's cached bounding volume, which is a problem if
  `calc_tight_bounds()` was itself invoked mid-traversal (e.g. by a
  `ShowBoundsEffect`); as a "hacky fix," it force-recomputes every direct
  parent's bounds immediately afterward.
- **`find_joint()`/`find_slider()` distinguish by runtime type, not just
  lookup success:** both call `PartGroup::find_child()` (recursive,
  name-based) and then check `is_character_joint()` or
  `is_of_type(CharacterSlider::get_class_type())` — a name that resolves to
  the wrong kind of part returns `nullptr` from the mismatched accessor.
- **`merge_bundles()` unifies two related skeletons** (e.g. different LODs
  of the same model sharing a common rig): it recursively walks both joint
  hierarchies in alphabetical order (`r_merge_bundles()`, mirroring
  `PartGroup::bind_hierarchy()`'s matching scheme), copying over any joints
  present in the old hierarchy but missing from the new one, and preserving
  `_net_transform_nodes`/`_local_transform_nodes` bindings from replaced
  joints onto their new counterparts. The two-argument `PartBundle*`
  overload is `@deprecated` in favor of the `PartBundleHandle*` overload.
- **Nested `Character` nodes get their own `copy_subgraph()` call** during
  deep-copy (`r_copy_char()`) rather than being flattened into the
  containing character's joint-redirection pass — a `Character` parented
  under another `Character` is copied independently.
- **The destructor clears every joint's back-pointer** to this `Character`
  via `r_clear_joint_characters()`, but only where `joint->get_character()
  == this` — after `merge_bundles()`, a joint can be listed under more than
  one `Character` but only points back to one, so a joint shared with a
  surviving `Character` is left alone.
- **Bam compatibility:** newer bam files no longer serialize an explicit
  array of parts (the joint hierarchy is discovered via the bundle instead),
  but `fillin()`/`complete_pointers()` still read and discard
  `_temp_num_parts` old-style pointers for backward compatibility with
  older files.

## API

### Construction & copying
| Signature | Notes |
|---|---|
| `explicit Character(const std::string &name)` | Also constructs a fresh `CharacterJointBundle(name)` |
| `virtual PandaNode *make_copy() const` | New joints + vertex arrays; drops geometry — see Behavior notes |
| `virtual PandaNode *dupe_for_flatten() const` | Shares internal data with the original; used by the flattener |
| `virtual PandaNode *combine_with(PandaNode *other)` | Merges two exact-type `Character`s via `steal_bundles()` |

### Bundle access
| Signature | Notes |
|---|---|
| `INLINE CharacterJointBundle *get_bundle(int i) const` | DCAST wrapper over `PartBundleNode::get_bundle()` |
| `void merge_bundles(PartBundle *old_bundle, PartBundle *other_bundle)` | `@deprecated`; see the `PartBundleHandle` overload |
| `void merge_bundles(PartBundleHandle *old_bundle_handle, PartBundleHandle *other_bundle_handle)` | Unifies two related joint hierarchies — see Behavior notes |

### LOD animation throttling
| Signature | Notes |
|---|---|
| `void set_lod_animation(const LPoint3 &center, PN_stdfloat far_distance, PN_stdfloat near_distance, PN_stdfloat delay_factor)` | Animate less often as camera distance grows past `near_distance` |
| `void clear_lod_animation()` | Undo the above; animate every frame again |

### Joint / slider lookup and inspection
| Signature | Notes |
|---|---|
| `CharacterJoint *find_joint(const std::string &name) const` | Recursive by-name search; never returns a slider |
| `CharacterSlider *find_slider(const std::string &name) const` | Recursive by-name search; never returns a joint |
| `void write_parts(std::ostream &out) const` | Dumps joint/slider hierarchy structure |
| `void write_part_values(std::ostream &out) const` | Same, plus each part's current value |

### Update
| Signature | Notes |
|---|---|
| `void update_to_now()` | `@deprecated` alias for `update()` |
| `void update()` | Recomputes joints/vertices once per frame; see Behavior notes |
| `void force_update()` | Recomputes unconditionally, bypassing the per-frame dedup |

## Usage

```cpp
#include "character.h"
#include "characterJointBundle.h"
#include "characterJoint.h"
#include "characterSlider.h"
#include "pandaNode.h"

PT(Character) actor = new Character("Actor");
CharacterJointBundle *bundle = actor->get_bundle(0);

CharacterJoint *right_hand = actor->find_joint("RightHand");
CharacterSlider *smile = actor->find_slider("smile");

// Attach a prop to the hand: it will track the joint's net transform
// every frame, and gets a CharacterJointEffect installed automatically.
if (right_hand != nullptr) {
  PT(PandaNode) sword = new PandaNode("sword");
  right_hand->add_net_transform(sword);
}

// Animate less often once the character is far from the camera.
actor->set_lod_animation(LPoint3(0, 0, 1), 50.0f, 10.0f, 0.2f);

// Normally cull_callback() calls this once per frame automatically.
actor->update();

if (smile != nullptr) {
  PN_stdfloat current_smile = smile->get_value();
}
```

## See also

[CharacterJointBundle](CharacterJointBundle.md), [CharacterJoint](CharacterJoint.md),
[CharacterSlider](CharacterSlider.md), [CharacterVertexSlider](CharacterVertexSlider.md),
[JointVertexTransform](JointVertexTransform.md), [CharacterJointEffect](CharacterJointEffect.md),
[../chan/PartBundleNode.md](../chan/PartBundleNode.md), [../chan/PartBundle.md](../chan/PartBundle.md),
[../chan/PartGroup.md](../chan/PartGroup.md), [README.md](README.md)
