# DrawCullHandler

**Source:** `panda/src/cull/drawCullHandler.{h,I,cxx}`
**Inherits from:** [`CullHandler`](../pgraph/CullHandler.md)

A `CullHandler` that draws each object immediately as the `CullTraverser`
discovers it, fusing the cull and draw traversals into one. Used when the
`Pipeline` has no separate cull/draw stage to hand work off to.

## Behavior

`record_object()`:
1. munges the object's geometry for the GSG's requirements via
   `object->munge_geom(gsg, gsg->get_geom_munger(object->_state,
   current_thread), traverser, force)`, where `force` is
   `!gsg->get_effective_incomplete_render()`;
2. if munging succeeded, draws the object immediately (same draw logic as the
   `CullBin` subclasses — state+transform set, then
   `GeomPipelineReader`/`GeomVertexDataPipelineReader` draw, or
   `draw_callback()` if the object has one);
3. `delete`s the `CullableObject` unconditionally afterward.

Because there's no bin, there is **no state sorting and no back-to-front
transparency ordering** — objects draw strictly in the order the traverser
visits them (scene-graph order). Trade-off: lower overhead (no allocation into
a bin, no second draw pass) at the cost of not being able to reorder for state
efficiency or transparency, and no ability to split cull and draw across
pipeline stages/threads.

## API reference

```cpp
DrawCullHandler(GraphicsStateGuardianBase *gsg);

virtual void record_object(CullableObject *object,
                            const CullTraverser *traverser);
```

- `DrawCullHandler(gsg)` — binds to the GSG objects will be drawn into. Holds
  a raw, non-owning `GraphicsStateGuardianBase *`.
- `record_object(object, traverser)` — munge, draw, delete, as above.

## Usage

Not constructed directly by application code — selected internally by the
render pipeline setup when single-stage (immediate) rendering is in effect.

## Related classes

- [`CullHandler`](../pgraph/CullHandler.md) — base class
- [`BinCullHandler`](BinCullHandler.md) — the binning alternative (state sort, transparency order, multi-stage)
- [`CullableObject`](../pgraph/CullableObject.md) — `munge_geom()` lives here
- [`CullTraverser`](../pgraph/CullTraverser.md) — calls `record_object()`
