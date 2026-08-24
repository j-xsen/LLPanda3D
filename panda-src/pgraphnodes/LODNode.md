# LODNode

**Source:** `panda/src/pgraphnodes/lodNode.h` (+ `.I`, `.cxx`; also folds in `lodNodeType.h`)
**Inherits:** `PandaNode` **Inherited by:** [FadeLODNode](FadeLODNode.md)

A Level-of-Detail node: it selects exactly one of its children for
rendering, based on the distance from the camera to a configurable center
point and a per-child in/out switch-distance range. It is used by parenting
one model variant per detail level as a child, then calling `add_switch()`
once per child (in the same order as the children) to define each level's
visible-distance range.

## `LODNodeType`

`LODNode::make_default_lod()` picks a concrete subclass based on the
`default-lod-type` config var (see [pgraphnodes README](README.md#config-variables-from-config_pgraphnodesh-cxx)):

| Value | Meaning |
|---|---|
| `LNT_pop` | Plain `LODNode` — instantaneous "pop" switch between levels |
| `LNT_fade` | [FadeLODNode](FadeLODNode.md) — cross-fades between levels over `lod-fade-time` seconds |

## Behavior notes

- **"In" vs. "out" is reversed from the intuitive guess.** Switch ranges are
  named as if the object were approaching from far away: a level switches
  **in** at the *far* (larger) distance and **out** at the *close* (smaller)
  distance, so `in` must be `>= out` for a given `add_switch(in, out)` call.
  A child is visible when `out <= dist < in`.
- **`cull_callback()` does its own child traversal and returns `false`** —
  it doesn't rely on `SelectiveChildNode`-style single-child selection
  (unlike [SwitchNode](SwitchNode.md)/[SequenceNode](SequenceNode.md)); it
  computes the camera-relative distance to `_center`, walks every switch
  range, and directly calls `trav->traverse()` on any child whose range
  contains that distance (normally exactly one, but nothing stops multiple
  or zero switch ranges from overlapping/gapping — getting the ranges right
  is the caller's responsibility).
- **Distance is computed relative to the `Camera`'s `lod_center`/
  `cull_center`, not the actual viewpoint**, if either is set (falls back to
  the ordinary modelview transform otherwise) — see `get_rel_transform()`.
  This allows LOD switching to be driven from a fixed reference point
  instead of a moving camera, e.g. for reproducible screenshots.
- **`get_lod_scale()`/`set_lod_scale()` multiplies the *comparison*
  distance**, not the switch ranges themselves — a higher scale makes
  levels switch at farther apparent distances (more detail shown farther
  away, i.e. lower overall performance). It's additionally multiplied by
  the `Camera`'s own `get_lod_scale()`, so both a per-node and a global
  scale apply.
- **`force_switch(index)`** overrides normal distance-based selection
  entirely, useful for debugging or forcing a specific detail level.
- **`verify_child_bounds()` / `verify-lods`:** if the `verify-lods` config
  var is true (debug builds only), every render checks that each child's
  actual bounding volume fits inside its switch-`in` radius sphere centered
  at `_center` — geometry poking out past its supposed switch-out distance
  is a common LOD-authoring bug this catches, raising an assertion
  (`nassert_raise`) with a suggested corrected radius.
- **`safe_to_combine()`/`safe_to_combine_children()` are both `false`** —
  scene graph flattening (`NodePath::flatten_strong()` etc., see
  [../pgraph/README.md](../pgraph/README.md)) must never merge an LODNode's
  children together or merge the LODNode itself into a parent, since the
  child list's order and identity are semantically meaningful (index-matched
  to the switch ranges).
- **`show_switch()`/`show_all_switches()` debug mode** replaces normal
  rendering with a wireframe visualization: a colored ring at each shown
  switch level's distance plus a vertical "spindle" through `_center`,
  procedurally generated as `Geom`s the first time they're needed and cached
  on the `Switch` struct.
- **`xform()` rescales switch distances**, using the length of the
  transform's Y-axis row as the scale factor — so non-uniform scaling of an
  LODNode applies an averaged/directional scale to distance thresholds,
  which can look wrong under strongly non-uniform scales.

## API

**Switch range management** (index-parallel to children — ranges must be
added in the same order children are parented):

| Method | Notes |
|---|---|
| `add_switch(in, out)` | Appends a new switch range; implies the corresponding child has been/will be parented at that index |
| `set_switch(index, in, out)` | Changes an existing range |
| `clear_switches()` | Removes all ranges |
| `get_num_switches()` / `get_in(index)` / `get_out(index)` / `get_ins()` / `get_outs()` | Accessors (`get_ins()`/`get_outs()` are `MAKE_SEQ` sequence properties) |
| `get_lowest_switch()` / `get_highest_switch()` | Index of the least-detailed (largest `out`) / most-detailed (smallest `in`) level |

**Distance/scale:**

| Method | Notes |
|---|---|
| `set_center(point)` / `get_center()` | The point (in LODNode-local space) compared against the camera |
| `set_lod_scale(value)` / `get_lod_scale()` | Multiplier on comparison distance; default `1` |
| `force_switch(index)` / `clear_force_switch()` | Force/unforce a specific level regardless of distance |

**Debug visualization:**

| Method | Notes |
|---|---|
| `show_switch(index)` / `show_switch(index, color)` | Enable ring+wireframe viz for one level |
| `hide_switch(index)` / `show_all_switches()` / `hide_all_switches()` | Bulk toggle |
| `is_any_shown()` | True if any level is in show mode |
| `verify_child_bounds()` | Manual on-demand bounds check (see Behavior notes) |

**Construction:**

| Method | Notes |
|---|---|
| `LODNode(name)` | Direct constructor — plain pop-switching |
| `LODNode::make_default_lod(name)` | Factory using `default-lod-type` to pick `LODNode` vs. `FadeLODNode` |

## Usage

```cpp
PT(LODNode) lod = new LODNode("building");
lod->add_child(high_detail_model);   // index 0
lod->add_child(low_detail_model);    // index 1
lod->add_switch(1000.0f, 0.0f);      // high detail: visible from 0 to 1000 units
lod->add_switch(1e9f, 1000.0f);      // low detail: visible from 1000 units on out
lod->set_center(LPoint3(0, 0, 0));
```

## See also

- [FadeLODNode](FadeLODNode.md) — cross-fading subclass
- [SwitchNode](SwitchNode.md), [SelectiveChildNode](SelectiveChildNode.md) — related "one visible child" nodes, but explicit-index rather than distance-driven
- [../pgraph/Camera.md](../pgraph/Camera.md) — `lod_center`/`cull_center`/`get_lod_scale()`
