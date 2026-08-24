# PartBundleNode

**Source:** `panda/src/chan/partBundleNode.h` / `.I` / `.cxx`
**Inherits:** `PandaNode`
**Inherited by:** `../char/Character.md`

A scene-graph node that holds one or more `PartBundle`s, wrapped in
[PartBundleHandle](PartBundleHandle.md)s. Like
[AnimBundleNode](AnimBundleNode.md), it exists to make storing bundles in the
scene graph possible — but unlike `AnimBundleNode`, it's also the base class
of the `char` module's `Character` node, which builds the actual animated
model on top of it.

## Behavior notes

- **Supports more than one bundle per node** (`_bundles` is a vector, not a
  single pointer) — `get_num_bundles()`/`get_bundle(n)` iterate it, though
  the common case (a `Character`) uses exactly one.
- **`add_bundle_handle()` de-duplicates by handle identity**, not by bundle
  content — calling it twice with the same handle is a no-op (checked via
  `find()`), but two different handles wrapping the *same* underlying
  `PartBundle` would both be added.
- **`~PartBundleNode()` calls `remove_node(this)` on every wrapped bundle** —
  this is the other half of [PartBundle::add_node()](PartBundle.md)'s
  bookkeeping, keeping `PartBundle::get_num_nodes()` accurate.
- **`apply_attribs_to_vertices()` is where transform-flattening on a
  character actually happens.** When the `SceneGraphReducer` is applying a
  `TT_transform` attrib, this calls
  [PartBundle::apply_transform()](PartBundle.md) per bundle (which returns a
  cached, already-transformed duplicate — see that page) and swaps it in via
  `update_bundle()`, then marks Geom bounds stale so they get recomputed.
  This is the mechanism that lets `flatten_strong()` on multiple `Actor`
  instances safely diverge their `PartBundle`s only when their applied
  transforms actually differ.
- **`xform()` (the older, simpler path) can't share bundles across nodes** —
  its doc comment explicitly recommends `apply_attribs_to_vertices()`
  instead. If `bundle->get_num_nodes() > 1` (this bundle is shared by more
  than one node), `xform()` first deep-copies the bundle via
  `copy_subgraph()` before transforming, so the transform doesn't leak into
  sibling nodes still sharing the original.
- **`steal_bundles()`** moves every bundle handle from another node onto
  this one (removing this node from each transferred bundle's owner list on
  the old node, adding it on the new one) — used during scene-graph
  restructuring where one node absorbs another's identity.

## API

### Bundle access
| Signature | Notes |
|---|---|
| `PartBundleNode(const std::string &name, PartBundle *bundle)` | Normal constructor; wraps `bundle` in a fresh `PartBundleHandle` |
| `int get_num_bundles() const` | Also `get_bundles`/`bundles` `MAKE_SEQ_PROPERTY` |
| `PartBundle *get_bundle(int n) const` | Unwraps handle `n` |
| `PartBundleHandle *get_bundle_handle(int n) const` | Also `get_bundle_handles`/`bundle_handles` `MAKE_SEQ_PROPERTY`; stable across flatten, unlike the raw bundle pointer |

### Internal management (protected — used by derived classes like Character)
| Signature | Notes |
|---|---|
| `void add_bundle(PartBundle *bundle)` | Wraps in a new handle and adds it |
| `void add_bundle_handle(PartBundleHandle *handle)` | De-duplicated by handle identity |
| `void steal_bundles(PartBundleNode *other)` | Transfers all bundles from `other` to `this` |
| `virtual void update_bundle(PartBundleHandle *old_bundle_handle, PartBundle *new_bundle)` | Swaps a handle's contents; overridden by `Character` |

### PandaNode overrides
| Signature | Notes |
|---|---|
| `virtual void apply_attribs_to_vertices(...)` | Applies an accumulated transform via bundle duplication; see Behavior notes |
| `virtual void xform(const LMatrix4 &mat)` | Simpler, non-sharing-aware transform path |

## Usage

```cpp
PartBundle *bundle = new PartBundle("skeleton");
PT(PartBundleNode) node = new PartBundleNode("actor", bundle);

NodePath render = window->get_render();  // WindowFramework* window
NodePath actor_np = render.attach_new_node(node);

int count = node->get_num_bundles();          // 1
PartBundle *same = node->get_bundle(0);        // == bundle
```

## See also

[PartBundle](PartBundle.md), [PartBundleHandle](PartBundleHandle.md),
[AnimBundleNode](AnimBundleNode.md), `../char/Character.md`, [README.md](README.md)
