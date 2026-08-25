# BinCullHandler

**Source:** `panda/src/cull/binCullHandler.{h,I,cxx}`
**Inherits from:** [`CullHandler`](../pgraph/CullHandler.md)

The standard `CullHandler` used for normal (non-immediate) rendering. Every
object the `CullTraverser` discovers is forwarded straight into a
[`CullResult`](../pgraph/CullResult.md), which sorts it into the correct
[`CullBin`](../pgraph/CullBin.md) for later drawing.

## Behavior

`record_object()` is a one-line forward: `_cull_result->add_object(object,
traverser)`. All of the actual binning/state-sort/transparency-order logic
lives in `CullResult` and the individual `CullBin` subclasses in this same
directory (see [`../cull/README.md`](README.md) for how bin choice is
decided). `BinCullHandler` itself holds no state beyond the `CullResult`
pointer it was constructed with.

This is the `CullHandler` a `CullTraverser` uses whenever cull and draw are
allowed to be separate stages (the normal, multi-stage `Pipeline`
configuration) — as opposed to [`DrawCullHandler`](DrawCullHandler.md), used
when they must be fused into one traversal.

## API reference

```cpp
BinCullHandler(CullResult *cull_result);

virtual void record_object(CullableObject *object,
                            const CullTraverser *traverser);
```

- `BinCullHandler(cull_result)` — wraps the given `CullResult`; does not take
  ownership (holds a raw `PT(CullResult)`, i.e. a reference-counted pointer,
  but does not create or destroy the `CullResult` itself).
- `record_object(object, traverser)` — forwards `object` to
  `cull_result->add_object(object, traverser)`.

## Usage

Not constructed directly by application code — `CullTraverser`/`SceneSetup`
set this up internally as part of the standard render-to-window path. Relevant
mainly when reading or extending the cull pipeline itself.

## Related classes

- [`CullHandler`](../pgraph/CullHandler.md) — base class
- [`CullResult`](../pgraph/CullResult.md) — where objects actually get sorted into bins
- [`DrawCullHandler`](DrawCullHandler.md) — the fused-traversal alternative
- [`CullTraverser`](../pgraph/CullTraverser.md) — calls `record_object()`
