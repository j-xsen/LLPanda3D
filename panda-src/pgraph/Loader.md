# Loader

**Source:** `panda/src/pgraph/loader.h` (+ `.I`, `.cxx`)
**Inherits:** `TypedReferenceCount`, `Namable`

The front-end for loading (and saving) model files. Get the shared
application-wide instance with `Loader::get_global_ptr()` (used implicitly
by `NodePath::load()`-style helpers elsewhere in the engine), or construct
your own if you want an isolated task chain/thread pool. Supports both
synchronous loads (`load_sync()`, blocks until done) and asynchronous loads
(`make_async_request()` + `load_async()`, runs on a background
[AsyncTaskChain](../event/AsyncTaskChain.md)).

## Behavior notes

- **Every `Loader` gets its own task chain**, named after the `Loader`'s own
  `Namable` name (default `"loader"`), created lazily in the constructor if
  it doesn't already exist on the global [AsyncTaskManager](../event/AsyncTaskManager.md).
  The chain is configured with `loader-num-threads` (default 1) worker
  threads and `loader-thread-priority` (default `low`). If threading support
  isn't compiled in, tasks on this chain still run, just synchronously when
  polled — the async interface degrades gracefully rather than failing.
- **Extension resolution:** if `filename` has no extension, the
  `default-model-extension` config var is appended. `.pz`/`.gz` suffixes are
  recognized and stripped for type lookup (compressed loading), but only if
  `HAVE_ZLIB` was compiled in and the resolved `LoaderFileType` reports
  `supports_compressed()`.
- **Search path:** `LoaderOptions::LF_search` (on by default via
  `LoaderOptions()`'s default flags) makes `load_sync()` search
  `get_model_path()` (the global model search path) for the file; it's
  automatically forced off for a non-local (absolute/already-rooted)
  filename.
- **Fallback-to-`.bam`:** if no `LoaderFileType` matches the requested
  extension (or the match fails to produce a result), `try_load_file()`
  retries by appending `.bam` to the path before giving up — so
  `load_sync("foo.egg")` transparently picks up a pre-converted
  `foo.egg.bam` if the original egg loader isn't available.
- **Two independent caches sit in front of disk I/O**, checked in this
  order inside `try_load_file()`: the in-RAM [ModelPool](ModelPool.md)
  (skippable with `LF_no_ram_cache`) and the on-disk `BamCache` (Panda's
  general compiled-asset cache, skippable with `LF_no_disk_cache`;
  `LF_no_cache` sets both, `LF_cache_only` fails outright rather than
  hitting disk when neither cache has it). A `ModelPool` hit returns a
  **shared** node unless `LF_allow_instance` is absent, in which case a deep
  copy (`NodePath::copy_to()`) is returned instead so callers can't
  accidentally mutate the cached original.
- **`premunge_data` (config var)** runs a [SceneGraphReducer](SceneGraphReducer.md)`::premunge()`
  pass over every freshly-loaded (non-cached) result before returning it.
- Async requests ([ModelLoadRequest](ModelLoadRequest.md)/[ModelSaveRequest](ModelSaveRequest.md))
  are plain [AsyncTask](../event/AsyncTask.md)s that call back into
  `load_sync()`/`save_sync()` on a worker thread — `load_async()` just
  stamps the request with this Loader's task chain name and adds it to the
  task manager.
- `remove()` is explicitly deprecated in favor of calling `task->cancel()`
  directly on the request (an `AsyncTask` is also an `AsyncFuture`).

## API

| Method | Notes |
|---|---|
| `explicit Loader(const std::string &name = "loader")` | Name doubles as the task chain name |
| `PT(PandaNode) load_sync(filename, options=LoaderOptions())` | Blocking load |
| `PT(AsyncTask) make_async_request(filename, options=LoaderOptions())` | Creates a `ModelLoadRequest`; doesn't start it |
| `void load_async(AsyncTask *request)` | Starts a request made by `make_async_request()` |
| `bool save_sync(filename, options, PandaNode *node)` | Blocking save |
| `PT(AsyncTask) make_async_save_request(filename, options, node)` | Creates a `ModelSaveRequest` |
| `void save_async(AsyncTask *request)` | Starts a save request |
| `PT(PandaNode) load_bam_stream(istream &in)` | Reads one `.bam`-format node directly from an arbitrary stream (bypasses `LoaderFileType` dispatch) |
| `void set_task_manager(AsyncTaskManager*)` / `get_task_manager()` | Default: global task manager |
| `void set_task_chain(const string&)` / `get_task_chain()` | Default: the Loader's name |
| `bool remove(AsyncTask*)` | Deprecated — use `task->cancel()` |
| `void stop_threads()` | Blocking; stops this Loader's task-chain threads |
| `static Loader *get_global_ptr()` | The shared application Loader |

**`Loader::Results`** (returned by directory-scanning helpers elsewhere,
e.g. `VirtualFileSystem` glob results fed through a loader): a simple
`(Filename, LoaderFileType*)` pair list — `get_num_files()`, `get_file(n)`,
`get_file_type(n)`, `add_file()`.

Key `LoaderOptions` flags (from `putil/loaderOptions.h`, used throughout):
`LF_search`, `LF_report_errors` (both on by default), `LF_convert_skeleton`,
`LF_convert_channels`, `LF_convert_anim` (= skeleton|channels),
`LF_no_disk_cache`, `LF_no_ram_cache`, `LF_no_cache` (= both), `LF_cache_only`,
`LF_allow_instance`.

## Usage

```cpp
Loader *loader = Loader::get_global_ptr();

// Synchronous:
PT(PandaNode) model = loader->load_sync("models/panda-model");
NodePath model_np(model);
model_np.reparent_to(render);

// Asynchronous:
PT(AsyncTask) req = loader->make_async_request("models/panda-model");
req->set_done_event("model_loaded");
loader->load_async(req);
// ... later, after the event fires:
// PT(PandaNode) model = ((ModelLoadRequest *)req.p())->get_model();
```

## See also

- [ModelPool](ModelPool.md), [BamFile](BamFile.md), [LoaderFileType](LoaderFileType.md),
  [LoaderFileTypeRegistry](LoaderFileTypeRegistry.md), [ModelLoadRequest](ModelLoadRequest.md),
  [ModelSaveRequest](ModelSaveRequest.md)
- [AsyncTask](../event/AsyncTask.md), [AsyncTaskManager](../event/AsyncTaskManager.md) (event module)
