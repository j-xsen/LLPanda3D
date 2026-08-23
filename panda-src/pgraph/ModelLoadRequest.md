# ModelLoadRequest

**Source:** `panda/src/pgraph/modelLoadRequest.h` / `.I` / `.cxx`
**Inherits:** [AsyncTask](../event/AsyncTask.md)

An [AsyncTask](../event/AsyncTask.md) that performs one asynchronous model
load. Created by [Loader](Loader.md)`::make_async_request()`; not usually
constructed directly.

## Behavior notes

- `do_task()` optionally sleeps `async-load-delay` seconds (a config var,
  useful for artificially testing async-load latency/UI), then calls
  `Loader::load_sync()` synchronously **on the worker thread** and stores
  the result via `set_result()` (the `AsyncFuture` mechanism inherited
  through `AsyncTask`), then returns `DS_done` — the task never
  reschedules itself.
- `is_ready()`/`get_model()` are a pre-`AsyncFuture` convenience API kept
  for compatibility; `get_model()` is explicitly marked deprecated in favor
  of the inherited `result()`. `get_model()` asserts `done()` is true.

## API

| Method | Notes |
|---|---|
| `explicit ModelLoadRequest(name, filename, options, Loader*)` | |
| `const Filename &get_filename() const` | |
| `const LoaderOptions &get_options() const` | |
| `Loader *get_loader() const` | |
| `bool is_ready() const` | `done() && !cancelled()` |
| `PandaNode *get_model() const` | Deprecated — use `result()` |

## Usage

```cpp
PT(AsyncTask) req = loader->make_async_request("models/panda-model");
req->set_done_event("model-loaded");
loader->load_async(req);
// on "model-loaded":
PT(PandaNode) model = (PandaNode *)((ModelLoadRequest *)req.p())->result();
```

## See also

- [Loader](Loader.md), [ModelSaveRequest](ModelSaveRequest.md), [ModelFlattenRequest](ModelFlattenRequest.md), [AsyncTask](../event/AsyncTask.md)
