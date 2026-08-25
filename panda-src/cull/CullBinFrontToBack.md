# CullBinFrontToBack

**Source:** `panda/src/cull/cullBinFrontToBack.{h,I,cxx}`
**Inherits from:** [`CullBin`](../pgraph/CullBin.md)

Sorts geometry nearest-to-furthest by distance from the camera to each
object's bounding-volume center — the mirror image of
[`CullBinBackToFront`](CullBinBackToFront.md). Intended for opaque geometry,
to take advantage of a hierarchical Z-buffer's early-out when a later object
is behind one already drawn.

## Behavior

Identical to `CullBinBackToFront`'s `add_object()` (empty bounds → drop the
object; otherwise compute `gsg->compute_distance_to(approx_center)` and store
it), except `finish_cull()`'s `ObjectData::operator<` is `_dist < other._dist`
— **ascending** distance, nearest object sorts first. One small source
difference from `CullBinBackToFront`: this class calls
`object->_geom->get_bounds()` (no explicit `current_thread` argument) rather
than `get_bounds(current_thread)`. `draw()` uses the same shared draw logic
(see [`../cull/README.md`](README.md#core-concepts)). The destructor deletes
every remaining object.

## API reference

```cpp
CullBinFrontToBack(const std::string &name, GraphicsStateGuardianBase *gsg,
                   const PStatCollector &draw_region_pcollector);

static CullBin *make_bin(const std::string &name,
                          GraphicsStateGuardianBase *gsg,
                          const PStatCollector &draw_region_pcollector);

virtual void add_object(CullableObject *object, Thread *current_thread);
virtual void finish_cull(SceneSetup *scene_setup, Thread *current_thread);
virtual void draw(bool force, Thread *current_thread);
```

Constructor forwards to `CullBin(name, BT_front_to_back, gsg,
draw_region_pcollector)`. `make_bin()` is registered against
`CullBinManager::BT_front_to_back` in `init_libcull()`.

## Usage

Selected via a `CullBinAttrib` naming a bin registered as `BT_front_to_back`.
Not constructed directly.

## Related classes

- [`CullBin`](../pgraph/CullBin.md) — base class
- [`CullBinManager`](../pgraph/CullBinManager.md) — bin registry / `cull-bin` config
- [`CullBinBackToFront`](CullBinBackToFront.md) — same distance metric, opposite order, for transparency
- [`CullBinStateSorted`](CullBinStateSorted.md) — groups by state instead of pure distance
