# CullBinStateSorted

**Source:** `panda/src/cull/cullBinStateSorted.{h,I,cxx}`
**Inherits from:** [`CullBin`](../pgraph/CullBin.md)

Sorts geometry to group objects with matching `RenderState` together, so the
GSG makes as few state changes as possible while drawing the bin. Within a
state group, additionally sorts front-to-back to take advantage of
hierarchical-Z early-out. Intended for opaque geometry (where draw order
doesn't affect correctness, only performance).

## Behavior

`add_object()` wraps each object in a private `ObjectData` (object pointer +
cached vertex format pointer) and appends it. `finish_cull()` runs
`std::sort` over `_objects` (timed by `_cull_this_pcollector`); `ObjectData::
operator<` defines the order, checked in this priority:

1. `RenderState::compare_sort()` between the two objects' `_state` — groups by
   state, "from heaviest change to lightest" per the source comment;
2. cached `GeomVertexFormat` pointer (format changes are "fairly slow");
3. `_munged_data` pointer ("prevent unnecessary vertex buffer rebinds");
4. `_internal_transform` pointer ("uniform updates are actually pretty
   fast" — lowest-priority tiebreak).

`draw()` then walks the now-sorted `_objects` using the shared draw logic
(see [`../cull/README.md`](README.md#core-concepts)). The destructor deletes
every remaining object.

## API reference

```cpp
CullBinStateSorted(const std::string &name, GraphicsStateGuardianBase *gsg,
                   const PStatCollector &draw_region_pcollector);

static CullBin *make_bin(const std::string &name,
                          GraphicsStateGuardianBase *gsg,
                          const PStatCollector &draw_region_pcollector);

virtual void add_object(CullableObject *object, Thread *current_thread);
virtual void finish_cull(SceneSetup *scene_setup, Thread *current_thread);
virtual void draw(bool force, Thread *current_thread);
```

Constructor forwards to `CullBin(name, BT_state_sorted, gsg,
draw_region_pcollector)`. `make_bin()` is registered against
`CullBinManager::BT_state_sorted` in `init_libcull()`.

## Usage

Selected via a `CullBinAttrib` naming a bin registered as `BT_state_sorted` —
this is the type of Panda's built-in `"opaque"` bin. Not constructed directly.

## Related classes

- [`CullBin`](../pgraph/CullBin.md) — base class
- [`CullBinManager`](../pgraph/CullBinManager.md) — bin registry / `cull-bin` config
- [`CullBinFrontToBack`](CullBinFrontToBack.md) — pure distance sort, no state grouping
