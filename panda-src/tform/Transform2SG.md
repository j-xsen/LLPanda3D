# Transform2SG

**Source:** `panda/src/tform/transform2sg.h` / `.cxx`
**Inherits:** `DataNode`

`Transform2SG` is the simplest possible data-graph consumer: it reads a
`transform` ([TransformState](../pgraph/TransformState.md)) input wire and
applies it, verbatim, as the transform on one designated `PandaNode`. It has
no outputs — it's a terminal leaf whose only effect is the side effect on
`_node`. This is the standard way to turn a [Trackball](Trackball.md)'s or
[DriveInterface](DriveInterface.md)'s computed matrix into an actual moved
node or camera.

## Behavior notes

- **A `Transform2SG` with no node set (`_node == nullptr`) is a silent
  no-op**, not an error — `do_transmit_data()` checks `_node != nullptr`
  before calling `set_transform()`; it still consumes the input each frame,
  it just discards it. Forgetting `set_node()` produces no diagnostic.
- **The transform is applied unconditionally when present, with no
  filtering, smoothing, or composition with the node's existing
  transform** — `_node->set_transform(transform, current_thread)` fully
  replaces whatever transform the node had. Anything needing to combine
  this with another transform (e.g. an offset) must do so upstream, before
  the `transform` wire reaches this node.
- **If the `transform` input isn't present this frame** (`input.has_data()`
  false — the producer upstream didn't set it), `_node`'s transform is left
  entirely untouched for that frame, i.e. it holds its last-set value
  rather than reverting to identity.
- **The current thread comes from the traverser, not a global lookup** —
  `do_transmit_data()` takes `trav->get_current_thread()` and passes it
  through to `set_transform()`, keeping the write on the correct pipeline
  stage for whichever thread is running this traversal.

## API

| Signature | Notes |
|---|---|
| `explicit Transform2SG(const std::string &name)` | Defines the `transform` input wire only; no outputs |
| `void set_node(PandaNode *node)` / `PandaNode *get_node() const` | The node whose transform this writes to; `nullptr` by default |

### DataNode override
| Signature | Notes |
|---|---|
| `virtual void do_transmit_data(...)` | Input: `transform`. No outputs — see behavior notes |

## Usage

```cpp
PT(Trackball) trackball = new Trackball("trackball");
NodePath tb_np = mak_np.attach_new_node(trackball);

PT(Transform2SG) xform = new Transform2SG("cam_xform");
xform->set_node(camera.node());
tb_np.attach_new_node(xform);  // camera now follows the trackball
```

## See also

[Trackball](Trackball.md), [DriveInterface](DriveInterface.md) (typical
upstream `transform` producers) ·
[TransformState](../pgraph/TransformState.md) (the value type carried) ·
[DataNode](../dgraph/DataNode.md) (base wire protocol)
