# CullBinFixed

**Source:** `panda/src/cull/cullBinFixed.{h,I,cxx}`
**Inherits from:** [`CullBin`](../pgraph/CullBin.md)

Sorts geometry by a user-specified integer draw order, giving precise manual
control over relative draw ordering between objects.

## Behavior

`add_object()` reads `object->_state->get_draw_order()` (i.e. the
`RenderState`'s draw-order attribute) and stores it with the object in
`ObjectData`. `finish_cull()` runs `std::stable_sort` (not `std::sort`, unlike
every other sorted bin in this module) over `_objects`, timed by
`_cull_this_pcollector`; `ObjectData::operator<` is `_draw_order <
other._draw_order` — ascending. Because the sort is *stable*, objects sharing
the same `draw_order` retain their original scene-graph order relative to each
other (matching `CullBinUnsorted`'s behavior for ties). `draw()` uses the
shared draw logic (see [`../cull/README.md`](README.md#core-concepts)). The
destructor deletes every remaining object.

## API reference

```cpp
CullBinFixed(const std::string &name, GraphicsStateGuardianBase *gsg,
             const PStatCollector &draw_region_pcollector);

static CullBin *make_bin(const std::string &name,
                          GraphicsStateGuardianBase *gsg,
                          const PStatCollector &draw_region_pcollector);

virtual void add_object(CullableObject *object, Thread *current_thread);
virtual void finish_cull(SceneSetup *scene_setup, Thread *current_thread);
virtual void draw(bool force, Thread *current_thread);
```

Constructor forwards to `CullBin(name, BT_fixed, gsg,
draw_region_pcollector)`. `make_bin()` is registered against
`CullBinManager::BT_fixed` in `init_libcull()`.

## Usage

Selected via a `CullBinAttrib` naming a bin registered as `BT_fixed` — this is
the type of Panda's built-in `"fixed"` bin (default draw order for things like
`background` and `gui-popup` in the standard bin config). Draw order is set
per-object via `NodePath::set_bin(bin_name, draw_order)`. Not constructed
directly.

## Related classes

- [`CullBin`](../pgraph/CullBin.md) — base class
- [`CullBinManager`](../pgraph/CullBinManager.md) — bin registry / `cull-bin` config
- [`CullBinUnsorted`](CullBinUnsorted.md) — same tie-breaking behavior, no primary sort key
