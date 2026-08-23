# ModelNode

**Source:** `panda/src/pgraph/modelNode.h` (+ `.I`, `.cxx`)
**Inherits:** PandaNode **Inherited by:** [ModelRoot](ModelRoot.md)

Marks the root of a "model" — a subtree conceptually treated as a single
unit (a car, a room). Doesn't affect rendering; it's a high-level
indication and a **set of flatten/transform protection flags**. Created
automatically in response to a `<Model> { 1 }` flag in an egg file.

## Behavior notes

- The whole class exists to answer `SceneGraphReducer`/flatten-related
  `PandaNode` virtuals based on the `PreserveTransform` enum
  (`_preserve_transform`, weakest → strongest):
  - `PT_none` (default): transform may be adjusted freely; node not removed.
  - `PT_net`: net transform from the root must be preserved, but the
    *local* transform on this node may be adjusted (avoids leaving an
    extra transform node above).
  - `PT_local`: local (and therefore net) transform must not change at
    all; the flattener will insert an extra transform node above this one
    if needed to absorb changes instead.
  - `PT_drop_node`: this `ModelNode` itself should be removed at the next
    flatten.
  - `PT_no_touch`: local transform frozen, node not removed, **and flatten
    does not recurse below this node at all** — the whole subtree is
    protected.
- Each virtual maps to a specific subset of these flags: `safe_to_flatten()`
  is true only for `PT_drop_node` (the node disappearing entirely is fine);
  `safe_to_flatten_below()` is true for everything except `PT_no_touch`;
  `safe_to_transform()` is true for `PT_none`/`PT_drop_node`;
  `safe_to_modify_transform()` is false only for `PT_local`/`PT_no_touch`;
  `safe_to_combine()` mirrors `safe_to_flatten()`; `preserve_name()` is
  false only for `PT_drop_node`/`PT_no_touch` (name is extrinsic
  bookkeeping there, not meaningful).
- `combine_with(other)`: if this node's flag is `PT_drop_node`, it
  unconditionally yields to `other` (i.e. this node vanishes, replaced by
  the other) rather than the usual `PandaNode::combine_with()` logic.
- `set_preserve_attributes(mask)` takes a bitmask of
  `SceneGraphReducer::AttribTypes` bits — attributes that must *not* be
  flattened onto this node's vertices; `get_unsafe_to_apply_attribs()`
  surfaces it back to the reducer (a generalization of
  `safe_to_transform()` to non-transform state).
- `set_transform_limit(limit)` + the overridden `transform_changed()` hook
  asserts (`nassertv`) every position component stays within
  `±_transform_limit` whenever the node's transform changes — a sanity
  check for catching runaway/garbage transforms during development; `0`
  (default) disables the check.

## API

| Method | Notes |
|---|---|
| `ModelNode(name)` | Constructor; `PreserveTransform` starts as `PT_none` |
| `set_preserve_transform(PreserveTransform)` / `get_preserve_transform()` | See flag table above |
| `set_preserve_attributes(int mask)` / `get_preserve_attributes()` | `SceneGraphReducer::AttribTypes` bits to protect |
| `set_transform_limit(float)` | Debug sanity-check bound on position components; `0` = off |

## Usage

```cpp
PT(ModelNode) mnode = new ModelNode("car");
mnode->set_preserve_transform(ModelNode::PT_local);
NodePath model_np = parent.attach_new_node(mnode);
```

## See also

- [ModelRoot](ModelRoot.md) — subclass specifically for Loader-loaded files
- [SceneGraphReducer](SceneGraphReducer.md) — the flatten operation these flags govern
