# Cull — Concrete Cull Bins and Cull Handlers

**Source:** `panda/src/cull/` · Library: `libp3cull` · Notify category: `cull`

This module has no framework classes of its own — `CullTraverser`, `CullHandler`,
`CullBin`, `CullBinManager`, `CullableObject`, `CullResult`, and `SceneSetup`
(the actual cull-traversal machinery) all live in
[`panda-src/pgraph/`](../pgraph/README.md). `cull` supplies the concrete,
leaf implementations of two of those abstract interfaces: two `CullHandler`
subclasses (what to do with each `CullableObject` as it's discovered) and five
`CullBin` subclasses (how to order objects for drawing once discovered). None
of the eight classes here is subclassed further within Panda3D.

This directory documents the public C++ API of every class in
`panda/src/cull`, for use without re-reading the engine source.

## Class map

```
CullHandler
├── BinCullHandler       (BinCullHandler.md)   — bins into a CullResult (normal path)
└── DrawCullHandler      (DrawCullHandler.md)  — draws immediately (fused cull+draw)

CullBin
├── CullBinUnsorted      (CullBinUnsorted.md)     — scene-graph order, no sort
├── CullBinStateSorted   (CullBinStateSorted.md)  — grouped by RenderState, for opaque geometry
├── CullBinBackToFront   (CullBinBackToFront.md)  — furthest-first, for transparency
├── CullBinFrontToBack   (CullBinFrontToBack.md)  — nearest-first, for hi-Z opaque geometry
└── CullBinFixed         (CullBinFixed.md)        — user-specified draw_order
```

`config_cull.h/.cxx` is the library's init file, not a class: `init_libcull()`
registers `TypeHandle`s for the five `CullBin` subclasses and registers each
one's `make_bin()` factory function with the global `CullBinManager` (keyed by
`CullBinManager::BinType` — `BT_unsorted`, `BT_state_sorted`,
`BT_back_to_front`, `BT_front_to_back`, `BT_fixed`). This is what makes
`cull-bin` config settings like `cull-bin foo 10 fixed` resolve to an actual
`CullBinFixed` instance. See
[`CullBinManager.md`](../pgraph/CullBinManager.md) in `pgraph` for the bin
registry itself (default bins, `cull-bin` config syntax).

## Core concepts

**`CullHandler::record_object()` is the fork in the road.** After
`CullTraverser` walks the scene graph and decides a node is visible and
renderable, it wraps it as a `CullableObject` and calls
`handler->record_object(object, traverser)`. Which `CullHandler` is plugged in
determines the rendering strategy for the whole frame:
- `BinCullHandler` — the default — forwards to `CullResult::add_object()`,
  which looks up the object's assigned `CullBin` (by its `RenderState`'s
  `CullBinAttrib`, falling back to bin `"default"`) and calls that bin's own
  `add_object()`. This is what enables state sorting, back-to-front
  transparency, and separating the cull and draw stages onto different
  pipeline stages/threads.
- `DrawCullHandler` — used when `Pipeline` has only one stage (no
  parallel cull-then-draw) — munges the geom for the GSG right away
  (`CullableObject::munge_geom()`) and draws it on the spot
  (`draw()`), then deletes the object immediately. No bin, no state
  sort, no back-to-front transparency ordering; lower overhead when those
  aren't needed.

**Every `CullBin` subclass here shares the same `draw()` body.** Once a bin's
objects are collected (and, for four of the five, sorted — see below), `draw()`
walks the collection and for each `CullableObject`:
- if `object->_draw_callback` is set, calls `object->draw_callback(gsg, force,
  current_thread)` and lets the callback do the drawing (used for e.g.
  decal/geom-node callbacks that need custom logic);
- otherwise calls `gsg->set_state_and_transform(object->_state,
  object->_internal_transform)` and draws `object->_geom` via a
  `GeomPipelineReader` + `GeomVertexDataPipelineReader` against
  `object->_munged_data`.

The five bins differ only in **what order** `_objects` ends up in, which is
decided in `add_object()` (what gets stored per-object) and `finish_cull()`
(the sort, called once after cull finishes and before draw starts):

| Bin | Sort key | Sort timing | Typical use |
|---|---|---|---|
| `CullBinUnsorted` | none | n/a | scene-graph order passthrough |
| `CullBinStateSorted` | `RenderState::compare_sort()`, then vertex format, then vertex data pointer, then transform pointer | `finish_cull()`, `std::sort` | opaque geometry — minimize GSG state changes |
| `CullBinBackToFront` | distance from GSG to bounding-volume center, descending | `finish_cull()`, `std::sort` | transparent/translucent geometry |
| `CullBinFrontToBack` | same distance, ascending | `finish_cull()`, `std::sort` | opaque geometry, hierarchical-Z early-out |
| `CullBinFixed` | `RenderState::get_draw_order()`, ascending | `finish_cull()`, `std::stable_sort` | precise manual layering; ties keep scene-graph order (stable) |

**Ownership.** Every bin and handler here takes ownership of the
`CullableObject *` pointers passed to `add_object()`/`record_object()` — they
`delete object` either right after drawing (`DrawCullHandler`) or in their own
destructor after `draw()` has run (all five `CullBin`s).

**PStats.** `finish_cull()` and `draw()` on every `CullBin` wrap their body in
a `PStatTimer` against `_cull_this_pcollector` / `_draw_this_pcollector`
(inherited from `CullBin`) — this is where per-bin cull/draw time shows up in
`pstats`.

## File index

| Class | Purpose |
|---|---|
| [BinCullHandler.md](BinCullHandler.md) | `CullHandler` that bins objects into a `CullResult` |
| [DrawCullHandler.md](DrawCullHandler.md) | `CullHandler` that draws objects immediately (fused cull+draw) |
| [CullBinUnsorted.md](CullBinUnsorted.md) | `CullBin`: no reordering, scene-graph order |
| [CullBinStateSorted.md](CullBinStateSorted.md) | `CullBin`: grouped by render state to minimize state changes |
| [CullBinBackToFront.md](CullBinBackToFront.md) | `CullBin`: furthest-to-nearest, for transparency |
| [CullBinFrontToBack.md](CullBinFrontToBack.md) | `CullBin`: nearest-to-furthest, for opaque hi-Z rendering |
| [CullBinFixed.md](CullBinFixed.md) | `CullBin`: explicit user-specified draw order |

## Status

cull — done (2026-08-25). Other `panda/src/*` subsystems not yet documented —
see `../../README.md` for the overall index.
