# PartBundleHandle

**Source:** `panda/src/chan/partBundleHandle.h` / `.I` / `.cxx`
**Inherits:** `ReferenceCount`

A trivial indirection wrapper returned by
[PartBundleNode::get_bundle_handle()](PartBundleNode.md). Its only job is to
hold the actual `PartBundle*`, so that scene-graph flatten operations (which
may combine or duplicate `PartBundle`s) can swap the pointer inside the
handle without invalidating anything holding onto the handle itself.

## Behavior notes

- **This exists specifically for `direct/src/actor`'s `Actor` class** (per
  the header comment): `Actor` stores `PartBundleHandle`s rather than raw
  `PartBundle*`s, so it stays valid across flatten operations that might
  replace the underlying bundle — see
  [PartBundleNode::update_bundle()](PartBundleNode.md), which is exactly
  the operation that swaps a handle's contents in place.
- The whole class is header-inline (`.cxx` is empty besides the include) —
  it's just a `PT(PartBundle)` with a getter/setter.

## API

| Signature | Notes |
|---|---|
| `PartBundleHandle(PartBundle *bundle)` | Wraps the given bundle |
| `PartBundle *get_bundle()` | The currently wrapped bundle |
| `void set_bundle(PartBundle *bundle)` | Swaps the wrapped bundle in place; also exposed as the `bundle` property |

## Usage

```cpp
PartBundle *bundle = new PartBundle("skeleton");
PartBundleNode *node = new PartBundleNode("actor", bundle);

PartBundleHandle *handle = node->get_bundle_handle(0);
PartBundle *same_bundle = handle->get_bundle();

// If a later flatten operation replaces the node's bundle, `handle` still
// points at whatever PartBundle is current -- code holding `handle` (not a
// raw PartBundle*) survives the swap.
```

## See also

[PartBundleNode](PartBundleNode.md), [PartBundle](PartBundle.md), [README.md](README.md)
