# AuxSceneData

**Source:** `panda/src/pgraph/auxSceneData.h` (+ `.I`)
**Inherits:** TypedReferenceCount — **not** a `PandaNode`

A base class for generic per-instance data attached to a
[`Camera`](Camera.md), used to persist state across multiple frames that
can't live on the scene graph node itself (since the same node may be
instanced under a camera multiple ways, or the data is inherently
per-camera). The motivating use case named in the header: `FadeLODNode`
needs to remember its current fade progress separately per instance *and*
per camera during traversal.

## Behavior notes

- The constructor is `protected` — you're expected to subclass and add
  actual payload data; a plain `AuxSceneData` carries nothing but the
  expiration bookkeeping below.
- **Expiration model**: `set_duration(seconds)` sets how long to keep the
  object around after it was last touched; `set_last_render_time(now)` is
  called each time it's used during traversal; `get_expiration_time()` is
  simply `_last_render_time + _duration`. [`Camera::cleanup_aux_scene_data()`](Camera.md)
  sweeps its `NodePath → AuxSceneData` map each call, comparing
  `get_expiration_time()` against the global clock and erasing anything
  past due — this class itself does no sweeping; it's purely passive data
  the camera manages.

## API

| Method | Notes |
|---|---|
| `set_duration(double)` / `get_duration()` | Seconds to retain after last use |
| `set_last_render_time(double)` / `get_last_render_time()` | Stamp each time the object is touched during traversal |
| `get_expiration_time()` | `last_render_time + duration` |

## Usage

```cpp
class MyAuxData : public AuxSceneData {
public:
  MyAuxData(double duration) : AuxSceneData(duration) {}
  double fade_progress = 0.0;
};

PT(MyAuxData) data = new MyAuxData(2.0);
camera->set_aux_scene_data(instance_node_path, data);
```

## See also

- [Camera](Camera.md) — owns and manages the `NodePath → AuxSceneData` cache
