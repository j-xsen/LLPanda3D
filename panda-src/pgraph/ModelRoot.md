# ModelRoot

**Source:** `panda/src/pgraph/modelRoot.h` (+ `.I`, `.cxx`)
**Inherits:** [ModelNode](ModelNode.md) > PandaNode

Created automatically at the root of each model file loaded by
[`Loader`](Loader.md). Currently carries no rendering-relevant data beyond
its base `ModelNode`; it mainly serves as a flag/marker that "a loaded
model file's root is here," plus bookkeeping for [`ModelPool`](ModelPool.md)'s
cache.

## Behavior notes

- `_fullpath`/`_timestamp` record where the model was loaded from and its
  file mtime at load time — used by `ModelPool` to detect a stale cache
  entry (source file changed since caching) versus a hit.
- `ModelReference` is a tiny separate `ReferenceCount` object
  (`get_reference()`/`set_reference()`) used to **unify references to the
  same underlying model** — multiple `ModelRoot` instances (e.g. from
  repeated instancing) can share one `ModelReference`, and
  `get_model_ref_count()` reports that shared refcount, distinct from the
  `ModelRoot` node's own PandaNode refcount.
- Inherits all `PreserveTransform` flatten-protection behavior from
  [ModelNode](ModelNode.md) unchanged — it doesn't override any of those
  virtuals itself.

## API

| Method | Notes |
|---|---|
| `ModelRoot(name)` / `ModelRoot(fullpath, timestamp)` | Constructors |
| `get_model_ref_count()` | Refcount of the shared `ModelReference`, not the node itself |
| `get_fullpath()` / `set_fullpath(Filename)` | Source file path |
| `get_timestamp()` / `set_timestamp(time_t)` | Source file mtime at load |
| `get_reference()` / `set_reference(ModelReference*)` | Shared-identity marker across instances of the same load |

## See also

- [ModelNode](ModelNode.md) — base class; all flatten-protection semantics
- [ModelPool](ModelPool.md) — caches loaded models keyed by path/timestamp
- [Loader](Loader.md) — creates `ModelRoot` at the top of every loaded file
