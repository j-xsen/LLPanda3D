# FadeLODNode

**Source:** `panda/src/pgraphnodes/fadeLodNode.h` (+ `.I`, `.cxx`; also folds in `fadeLodNodeData.h`)
**Inherits:** [LODNode](LODNode.md) > `PandaNode`

An [LODNode](LODNode.md) that cross-fades between detail levels instead of
popping instantaneously. When the selected switch level changes, the old
and new children are both rendered simultaneously for `get_fade_time()`
seconds, with the old one alpha-fading out and the new one alpha-fading in.
`LODNode::make_default_lod()` (with `default-lod-type` set to `LNT_fade`)
selects it, or it is constructed directly when the fade behavior is
specifically wanted.

## `FadeLODNodeData`

Per-instance fade progress can't live directly on the node, because the
same `FadeLODNode` can be instanced under multiple `NodePath`s (each
potentially at a different camera-relative distance, each needing its own
independent fade state) and even a single instance is watched by
potentially multiple cameras. It's tracked instead through `Camera`'s
generic `AuxSceneData` mechanism (see
[../pgraph/AuxSceneData.md](../pgraph/AuxSceneData.md)) — one
`FadeLODNodeData` per `(Camera, NodePath)` pair, expiring automatically if
not touched for a while (`camera->get_aux_scene_data()` returns `nullptr`
past `get_expiration_time()`, causing a fresh fade-free state).

`FadeLODNodeData` fields:

| Field | Meaning |
|---|---|
| `_fade_mode` | `FM_solid` (no transition in progress), `FM_more_detail`, or `FM_less_detail` |
| `_fade_start` | Frame time the current transition began |
| `_fade_out` / `_fade_in` | Switch indices of the outgoing/incoming child |

## Behavior notes

- **Fade direction is content-aware, not just "old vs. new."** When a
  transition starts, `cull_callback()` compares the new switch level's
  `out` distance against the old one's: switching to a level with a
  *larger* `out` (less detail) sets `_fade_mode = FM_less_detail`, and the
  fade timeline is played in reverse (`elapsed = _fade_time - elapsed`,
  in/out children swapped) "for best visual quality" per a source comment —
  the intent is that the more-detailed model's fade curve looks the same
  whether approaching or receding, rather than mirroring awkwardly.
- **Fade start time uses the *last-rendered* frame time, not "now".** If
  the node was off-screen for a while before a level change is first
  noticed, the fade is computed as if it started back then — so returning
  on-screen mid-transition shows only the tail of the fade rather than
  restarting it, avoiding a visible "pop" of a fresh fade beginning late.
- **Two-phase alpha blend, not a simple linear cross-dissolve:** during the
  first half of `_fade_time`, the old child renders opaque (depth writes
  on) while the new child fades in with depth writes *off*; during the
  second half, the new child snaps to fully opaque (depth writes on) while
  the old child fades out with depth writes off. This avoids
  z-fighting/depth-sorting artifacts between two half-transparent
  overlapping models, at the cost of a brief visible "seam" at the
  halfway point where the roles swap.
- **`set_fade_bin()`/`set_fade_state_override()` invalidate cached
  `RenderState`s.** The four fade-transition `RenderState`s
  (`_fade_1_old_state` etc.) are computed lazily and cached; changing the
  fade bin or override level clears the relevant cached states so they're
  rebuilt with the new settings on next use.
- **If `support-fade-lod` is globally false, `cull_callback()` degrades to
  plain `LODNode::cull_callback()`** — an instant pop, no fade — allowing
  fading to be disabled engine-wide (e.g. for a performance mode) without
  touching per-node settings.
- Only `LODNode::write_datagram()`/`fillin()` are actually called for bam
  persistence — `_fade_time`/`_fade_bin_name`/etc. are **not** currently
  serialized to `.bam` files (re-initialized from the `lod-fade-*` config
  vars on load instead); relevant when non-default fade parameters are set
  and expected to round-trip through a saved model.

## API

| Method | Notes |
|---|---|
| `FadeLODNode(name)` | Constructor; initializes fields from `lod-fade-time`/`lod-fade-bin-name`/`lod-fade-bin-draw-order`/`lod-fade-state-override` config vars |
| `set_fade_time(t)` / `get_fade_time()` | Transition duration in seconds |
| `set_fade_bin(name, draw_order)` / `get_fade_bin_name()` / `get_fade_bin_draw_order()` | `CullBinAttrib` bin the fading (transparent) geometry is sorted into during a transition |
| `set_fade_state_override(override)` / `get_fade_state_override()` | `RenderState` attrib override level applied to force the fade attribs to win; should exceed any override already on the fading geometry |

All switch-range/center/scale API is inherited unchanged from
[LODNode](LODNode.md).

## Usage

```cpp
PT(FadeLODNode) lod = new FadeLODNode("building");
lod->add_child(high_detail_model);
lod->add_child(low_detail_model);
lod->add_switch(1000.0f, 0.0f);
lod->add_switch(1e9f, 1000.0f);
lod->set_fade_time(0.5f);
```

## See also

- [LODNode](LODNode.md) — base class, switch-range/center/scale API
- [../pgraph/AuxSceneData.md](../pgraph/AuxSceneData.md) — per-camera
  instance data mechanism backing `FadeLODNodeData`
- [../pgraph/README.md](../pgraph/README.md) — `RenderState`/`CullBinAttrib`
