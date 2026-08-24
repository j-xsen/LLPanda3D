# SequenceNode

**Source:** `panda/src/pgraphnodes/sequenceNode.h` (+ `.I`, `.cxx`)
**Inherits:** [SelectiveChildNode](SelectiveChildNode.md) > `PandaNode`, `AnimInterface` (external, `chan` — undocumented)

Automatically cycles through rendering exactly one child at a time, one
child per "frame" of an animation clock — the scene-graph equivalent of a
flipbook. Each child is one frame; `get_num_frames()` returns
`get_num_children()` directly. All frame-rate/looping/play-range control comes from
the inherited `AnimInterface` (`chan` module, undocumented here — think
`play()`/`loop()`/`pose()`/`set_frame_rate()`-style controls).

## Behavior notes

- **The currently-visible child is determined solely by `get_frame()`** (from
  `AnimInterface`) — `cull_callback()` calls `select_child(get_frame())`
  each traversal, so the visible child tracks the animation clock
  automatically; there's no separate index state to keep in sync (unlike
  [SwitchNode](SwitchNode.md), which mirrors its own `CData`-stored index
  into the base class's selection each frame).
- **`has_single_child_visibility()` returns `true`**, same as `SwitchNode`
  — the visible child (`get_frame()`) is determined without external
  context, so `get_visible_child()` works directly.
- **`get_num_frames()` is virtual and derived from children**, so adding or
  removing children changes the animation's frame count live — there's no
  separately-stored frame count to fall out of sync.
- **`safe_to_combine()`/`safe_to_combine_children()` are both `false`** —
  same reasoning as `LODNode`/`SwitchNode`: child order is semantically
  meaningful (it's the frame sequence).
- Persistence combines two independent serialization calls:
  `SelectiveChildNode::write_datagram()`/`fillin()` for the node itself,
  then `AnimInterface::write_datagram()`/`fillin()` for the clock/frame-rate
  state — multiple inheritance, two separate datagram sections.

## API

Inherits all frame-rate/play-state control from `AnimInterface` (external,
undocumented). `SequenceNode`-specific additions:

| Method | Notes |
|---|---|
| `SequenceNode(name)` | Constructor |
| `get_num_frames()` | Virtual override; returns `get_num_children()` |
| `set_frame_rate(rate)` | `MAKE_PROPERTY frame_rate`; forwards to `AnimInterface` |

## Usage

```cpp
PT(SequenceNode) seq = new SequenceNode("flipbook");
seq->add_child(frame0);
seq->add_child(frame1);
seq->add_child(frame2);
seq->set_frame_rate(12.0);  // AnimInterface
seq->loop(true);            // AnimInterface
```

## See also

- [SelectiveChildNode](SelectiveChildNode.md) — base class
- [SwitchNode](SwitchNode.md) — explicit-index sibling
