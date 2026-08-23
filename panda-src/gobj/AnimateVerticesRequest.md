# AnimateVerticesRequest

**Source:** `panda/src/gobj/animateVerticesRequest.h` (+ `.I`, `.cxx`)
**Inherits:** [AsyncTask](../event/AsyncTask.md)

A background-thread task wrapper around
`GeomVertexData::animate_vertices()` — the CPU-skinning/morph-blend compute
step for one `GeomVertexData` (see
[TransformBlendTable](TransformBlendTable.md)/[SliderTable](SliderTable.md)).
Submitting one of these to an `AsyncTaskManager` task chain lets vertex
animation for multiple characters be computed in parallel across CPU cores
ahead of when they're actually needed for rendering. See
[../event/README.md](../event/README.md) for the general `AsyncTask`/
`AsyncTaskManager` task-chain system this builds on — this doc covers only
what's specific to this task.

## Behavior notes

- Holds no result and returns none: `do_task()` calls
  `_geom_vertex_data->animate_vertices(true, current_thread)` purely for
  its side effect — `GeomVertexData` caches the animated result
  internally, and later rendering code retrieves it through the normal
  `GeomVertexData` API. The comment stresses that the *caller* is
  responsible for making sure the main thread blocks until this task
  completes (e.g. by waiting on the task chain) before that render happens,
  since nothing else enforces the ordering.
  the `force=true` argument to `animate_vertices()` means this call blocks
  (paging in vertex data from disk if necessary) rather than returning
  immediately if data isn't resident — appropriate since it's already
  running off the main thread.
- `do_task()` always returns `AsyncTask::DS_done` — this is a one-shot
  task, not a recurring one.
- `is_ready()` — a thin status query exposed for callers that want to poll
  rather than block on the task chain directly.
- Uses `ALLOC_DELETED_CHAIN` for fast pooled allocation/deallocation, since
  these are typically created and destroyed in bulk once per frame (one
  per animated character).

## API

| Method | Notes |
|---|---|
| `explicit AnimateVerticesRequest(GeomVertexData *vertex_data)` | Wraps one `GeomVertexData` to animate. |
| `is_ready() const` | Whether the task has completed. |
| `do_task()` (protected, override) | Calls `animate_vertices(true, ...)`; always returns `DS_done`. |

## Usage

```cpp
PT(AnimateVerticesRequest) req = new AnimateVerticesRequest(geom_vertex_data);
AsyncTaskManager::get_global_ptr()->add(req);
// ... later, before rendering depends on the result:
AsyncTaskManager::get_global_ptr()->wait_for_tasks();
```

## See also

- [AsyncTask](../event/AsyncTask.md), [AsyncTaskManager](../event/AsyncTaskManager.md) — the task-chain system this runs on
- [GeomVertexData](GeomVertexData.md) — owns `animate_vertices()` and the cached result
- [TextureReloadRequest](TextureReloadRequest.md) — a similarly-shaped async task for texture loading
