# AnimBundleNode

**Source:** `panda/src/chan/animBundleNode.h` / `.I` / `.cxx`
**Inherits:** `PandaNode`

A thin scene-graph node that exists solely to hold a pointer to an
[AnimBundle](AnimBundle.md), so animation data can be stored, saved, and
loaded as part of an ordinary scene graph (e.g. inside a `.bam` file
alongside a model). Mirrors [PartBundleNode](PartBundleNode.md) on the
model side.

## Behavior notes

- **`safe_to_flatten()` always returns `false`.** Like a `Camera` node, an
  `AnimBundleNode`'s identity matters — flattening (which duplicates nodes
  by instance) would break the association between the node and its bundle.
- **`find_anim_bundle()` is a static free-standing search helper**, not an
  instance method: it recursively walks a scene graph from any given root
  and returns the first `AnimBundle` it finds on any `AnimBundleNode`,
  or `nullptr` if none exists. This is the common way to pull a loaded
  animation's bundle back out of a model/anim `.bam` file after loading.
- **The bundle pointer participates in the bam pointer-completion system**
  (`write_pointer`/`read_pointer`/`complete_pointers`), so an `AnimBundle`
  saved this way is properly restored on load rather than requiring manual
  reattachment.

## API

| Signature | Notes |
|---|---|
| `explicit AnimBundleNode(const std::string &name, AnimBundle *bundle)` | Normal constructor |
| `AnimBundle *get_bundle() const` | Property `bundle` |
| `static AnimBundle *find_anim_bundle(PandaNode *root)` | Recursive scene-graph search; `nullptr` if not found |
| `virtual bool safe_to_flatten() const` | Always `false` |
| `virtual PandaNode *make_copy() const` | |

## Usage

```cpp
#include "animBundle.h"
#include "animBundleNode.h"
#include "loader.h"
#include "nodePath.h"

Loader loader;
PT(PandaNode) anim_model = loader.load_sync("walk.bam");
if (anim_model != nullptr) {
  AnimBundle *bundle = AnimBundleNode::find_anim_bundle(anim_model);
  if (bundle != nullptr) {
    NodePath anim_np(anim_model);
    // bundle now ready to pass to PartBundle::bind_anim() or auto_bind().
  }
}
```

## See also

- [AnimBundle](AnimBundle.md) — the data this node wraps
- [PartBundleNode](PartBundleNode.md) — the equivalent wrapper on the model side
- [auto_bind](auto_bind.md) — commonly used to find and bind an
  `AnimBundleNode`'s bundle to a model in one call
- [README.md](README.md)
