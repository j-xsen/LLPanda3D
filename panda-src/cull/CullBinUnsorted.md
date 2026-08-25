# CullBinUnsorted

**Source:** `panda/src/cull/cullBinUnsorted.{h,I,cxx}`
**Inherits from:** [`CullBin`](../pgraph/CullBin.md)

The simplest `CullBin`: does not reorder geometry at all. Objects draw in the
same order the `CullTraverser` encountered them — scene-graph order.

## Behavior

`add_object()` just appends to an internal `pvector<CullableObject *>
_objects`. There is no `finish_cull()` override (no sort step). `draw()` walks
`_objects` front-to-back using the shared draw logic described in
[`../cull/README.md`](README.md#core-concepts) (state+transform set and geom
draw, or `draw_callback()`). The destructor deletes every remaining
`CullableObject *` in `_objects`.

Use this bin when draw order genuinely doesn't matter and scene-graph order is
already acceptable (or intentional) — it's the cheapest bin since it skips
sorting entirely.

## API reference

```cpp
CullBinUnsorted(const std::string &name, GraphicsStateGuardianBase *gsg,
                const PStatCollector &draw_region_pcollector);

static CullBin *make_bin(const std::string &name,
                          GraphicsStateGuardianBase *gsg,
                          const PStatCollector &draw_region_pcollector);

virtual void add_object(CullableObject *object, Thread *current_thread);
virtual void draw(bool force, Thread *current_thread);
```

- Constructor forwards to `CullBin(name, BT_unsorted, gsg,
  draw_region_pcollector)`.
- `make_bin(...)` — factory function; registered against
  `CullBinManager::BT_unsorted` in `init_libcull()` (see
  [`../cull/README.md`](README.md)).
- `add_object()` — appends, no sorting.
- `draw()` — draws in insertion order.

## Usage

Selected via `CullBinManager` when a `CullBinAttrib` on the render state names
a bin registered as `BT_unsorted` (the default `"fixed"`/`"opaque"`/etc. bins
each have their own configured type — see
[`CullBinManager.md`](../pgraph/CullBinManager.md)). Not constructed directly.

## Related classes

- [`CullBin`](../pgraph/CullBin.md) — base class
- [`CullBinManager`](../pgraph/CullBinManager.md) — bin registry / `cull-bin` config
- [`CullBinStateSorted`](CullBinStateSorted.md), [`CullBinFixed`](CullBinFixed.md) — sorted alternatives
