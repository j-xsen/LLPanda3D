# ModelSaveRequest

**Source:** `panda/src/pgraph/modelSaveRequest.h` / `.I` / `.cxx`
**Inherits:** [AsyncTask](../event/AsyncTask.md)

An [AsyncTask](../event/AsyncTask.md) that performs one asynchronous model
save. Created by [Loader](Loader.md)`::make_async_save_request()`.

## Behavior notes

- Mirrors [ModelLoadRequest](ModelLoadRequest.md): `do_task()` optionally
  sleeps `async-load-delay`, then calls `Loader::save_sync()` on the worker
  thread, stores the resulting bool in `_success` (not via `set_result()`
  — retrieved through `get_success()`, not `result()`), and returns
  `DS_done`.

## API

| Method | Notes |
|---|---|
| `explicit ModelSaveRequest(name, filename, options, PandaNode *node, Loader*)` | |
| `const Filename &get_filename() const` | |
| `const LoaderOptions &get_options() const` | |
| `PandaNode *get_node() const` | The node passed to the constructor |
| `Loader *get_loader() const` | |
| `bool is_ready() const` | `done() && !cancelled()` |
| `bool get_success() const` | Asserts `done()` |

## See also

- [Loader](Loader.md), [ModelLoadRequest](ModelLoadRequest.md), [AsyncTask](../event/AsyncTask.md)
