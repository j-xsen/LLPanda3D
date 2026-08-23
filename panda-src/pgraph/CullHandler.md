# CullHandler

**Source:** `panda/src/pgraph/cullHandler.h` (+ `.I`, `.cxx`)
**Inherited by:** `CullResult` (the standard/only substantial implementation shipped in `pgraph`)

Abstract callback interface that receives one `CullableObject` for every Geom the [CullTraverser](CullTraverser.md) finds visible during a traversal. By itself it just logs each object and deletes it (the default `record_object()`); real behavior comes from overriding it — [CullResult](CullResult.md) is the standard override that sorts objects into [CullBin](CullBin.md)s.

## Behavior notes

- **Ownership transfer:** `record_object()` takes ownership of the `CullableObject*` passed in and is responsible for `delete`-ing it eventually (the default implementation deletes it immediately after printing it via `nout`). Any custom `CullHandler` must not leak these.
- `end_traverse()` is a no-op hook called once after the whole traversal completes — `CullResult` uses it to finish/sort bins.
- The static inline `draw()` helper is a thin wrapper that just calls `object->draw(gsg, force, current_thread)` then deletes the object — a convenience for handlers that want to draw immediately instead of binning (e.g. some direct/immediate-mode paths).

## API

| Signature | Notes |
|---|---|
| `virtual void record_object(CullableObject *object, const CullTraverser *traverser)` | override point; default: print + delete |
| `virtual void end_traverse()` | override point; default: no-op |
| `static void draw(CullableObject *object, GraphicsStateGuardianBase *gsg, bool force, Thread *current_thread)` | draw-immediately convenience, deletes object |

## Usage

Application code virtually never implements `CullHandler` directly — `CullTraverser::traverse()` is invoked with a `CullHandler*`, and `CullResult` (created internally by `GraphicsEngine`/`DisplayRegion` rendering, see [display](../display/README.md)) is the one supplied in practice.

## See also

- [CullTraverser](CullTraverser.md) — drives calls into `record_object()`
- [CullResult](CullResult.md) — the standard implementation, sorts into bins
- [CullableObject](CullableObject.md) — the object type passed to `record_object()`
