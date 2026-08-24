# AnimBundle

**Source:** `panda/src/chan/animBundle.h` / `.I` / `.cxx`
**Inherits:** [AnimGroup](AnimGroup.md)
**Inherited by:** none (concrete leaf; wrapped in the scene graph by [AnimBundleNode](AnimBundleNode.md))

The root of an `AnimGroup` hierarchy. Every channel hierarchy needs exactly
one of these at its root — it's the only place frame rate and frame count
are recorded, and every channel beneath it is expected to agree with those
values.

## Behavior notes

- **`_root = this` is set in the constructor** — every descendant `AnimGroup`
  copies this pointer down at construction time (see
  [AnimGroup](AnimGroup.md) behavior notes), so the whole tree can find its
  bundle without a tree walk.
- **`get_base_frame_rate()` is the *ideal* rate, not necessarily the rate
  actually played.** The header is explicit: playback can run faster or
  slower via `AnimControl::set_play_rate()`; see
  `AnimControl::get_effective_frame_rate()` on [AnimControl](AnimControl.md)
  for the actual current rate.
- **`get_num_frames()` returns `0` for an animation with no fixed frame
  count** (e.g. a purely procedural/dynamic channel that doesn't loop over a
  table).
- **`copy_bundle()` deep-copies the group structure, but leaf data tables are
  shared, not duplicated** — documented directly in the `.cxx`. Two bundles
  produced this way can safely have independent hierarchies (e.g. for
  per-instance channel swapping) while still sharing the bulk memory of any
  `AnimChannelMatrixXfmTable` data.

## API

| Signature | Notes |
|---|---|
| `explicit AnimBundle(const std::string &name, PN_stdfloat fps, int num_frames)` | Normal constructor; sets itself as its own `_root` |
| `double get_base_frame_rate() const` | Property `base_frame_rate`; ideal fps, see notes |
| `int get_num_frames() const` | Property `num_frames`; `0` if no fixed frame count |
| `PT(AnimBundle) copy_bundle() const` | Deep-copies structure; leaf data tables shared |
| `virtual void output(std::ostream&) const` | e.g. `"AnimBundle walk, 48 frames at 24 fps"` |

## Usage

```cpp
#include "animBundle.h"
#include "animBundleNode.h"
#include "animChannelMatrixXfmTable.h"

PT(AnimBundle) bundle = new AnimBundle("walk", 24.0, 48);
PT(AnimChannelMatrixXfmTable) hip_channel =
  new AnimChannelMatrixXfmTable(bundle, "hip");

// Duplicate the hierarchy (e.g. per-instance); channel data is shared.
PT(AnimBundle) bundle_copy = bundle->copy_bundle();

NodePath anim_np("walk-anim");
PT(AnimBundleNode) anim_node = new AnimBundleNode("walk", bundle);
anim_np.node()->add_child(anim_node);
```

## See also

- [AnimGroup](AnimGroup.md) — the base class implementing the tree itself
- [AnimBundleNode](AnimBundleNode.md) — scene-graph wrapper for a bundle
- [AnimControl](AnimControl.md) — controls actual playback rate/time of a
  bound bundle
- [PartBundle](PartBundle.md) — the matching hierarchy on the model side
- [README.md](README.md)
