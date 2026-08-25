# CullBinBackToFront

**Source:** `panda/src/cull/cullBinBackToFront.{h,I,cxx}`
**Inherits from:** [`CullBin`](../pgraph/CullBin.md)

Sorts geometry furthest-to-nearest by distance from the camera to each
object's bounding-volume center. Intended for transparent/semi-transparent
geometry, which must draw back-to-front for correct blending.

## Behavior

`add_object()` computes `object->_geom->get_bounds(current_thread)`; if the
volume is empty, the object is dropped (`delete object`) without being added.
Otherwise it takes `GeometricBoundingVolume::get_approx_center()`, transforms
it by `object->_internal_transform->get_mat()`, and computes
`gsg->compute_distance_to(center)`, storing that distance with the object in
`ObjectData`. `finish_cull()` runs `std::sort` (timed by
`_cull_this_pcollector`); `ObjectData::operator<` is `_dist > other._dist` —
**descending** distance, i.e. furthest object sorts first. `draw()` walks the
sorted list with the shared draw logic (see
[`../cull/README.md`](README.md#core-concepts)). The destructor deletes every
remaining object.

## API reference

```cpp
CullBinBackToFront(const std::string &name, GraphicsStateGuardianBase *gsg,
                   const PStatCollector &draw_region_pcollector);

static CullBin *make_bin(const std::string &name,
                          GraphicsStateGuardianBase *gsg,
                          const PStatCollector &draw_region_pcollector);

virtual void add_object(CullableObject *object, Thread *current_thread);
virtual void finish_cull(SceneSetup *scene_setup, Thread *current_thread);
virtual void draw(bool force, Thread *current_thread);
```

Constructor forwards to `CullBin(name, BT_back_to_front, gsg,
draw_region_pcollector)`. `make_bin()` is registered against
`CullBinManager::BT_back_to_front` in `init_libcull()`.

## Usage

Selected via a `CullBinAttrib` naming a bin registered as `BT_back_to_front` —
this is the type of Panda's built-in `"transparent"` bin, which is what
`set_transparency()` on a `NodePath` typically routes geometry into. Not
constructed directly.

## Related classes

- [`CullBin`](../pgraph/CullBin.md) — base class
- [`CullBinManager`](../pgraph/CullBinManager.md) — bin registry / `cull-bin` config
- [`CullBinFrontToBack`](CullBinFrontToBack.md) — same distance metric, opposite order, for opaque geometry
