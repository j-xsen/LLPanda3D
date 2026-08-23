# AudioLoadRequest

**Source:** `panda/src/audio/audioLoadRequest.h` / `.I` / `.cxx`
**Inherits:** [AsyncTask](../event/AsyncTask.md)

An [AsyncTask](../event/AsyncTask.md) that performs one asynchronous sound
load. Despite the header comment mentioning it works "in conjunction with
the Loader class defined in pgraph," it has no actual dependency on
[Loader](../pgraph/Loader.md) — it runs on any `AsyncTaskManager`. Create one
and add it to a task manager to begin an async load.

## Behavior notes

- `do_task()` calls `_audio_manager->get_sound(_filename, _positional)`
  synchronously (on whatever thread runs the task) and stores the result via
  `set_result()` (the `AsyncFuture` mechanism inherited through `AsyncTask`),
  then unconditionally returns `DS_done` — the task never reschedules
  itself, unlike `pgraph`'s `ModelLoadRequest`, which supports an artificial
  `async-load-delay` for testing.
- `is_ready()`/`get_sound()` are a pre-`AsyncFuture` convenience API kept for
  compatibility; `get_sound()` is explicitly `@deprecated` in favor of the
  inherited `result()`.
- Uses `ALLOC_DELETED_CHAIN(AudioLoadRequest)` pooled allocation, same
  pattern as other lightweight task/request classes in the engine.
- Constructed with `AudioManager*`, filename, and `positional` flag — no
  `StreamMode` parameter, so it always requests `SM_heuristic` mode via
  `AudioManager::get_sound()`.

## API

| Signature | Notes |
|---|---|
| `explicit AudioLoadRequest(AudioManager*, const std::string &filename, bool positional)` | |
| `AudioManager *get_audio_manager() const` | |
| `const std::string &get_filename() const` | |
| `bool get_positional() const` | |
| `bool is_ready() const` | Pre-`AsyncFuture` convenience; prefer `done()` |
| `AudioSound *get_sound() const` | Deprecated — use `result()` |

## Usage

```cpp
PT(AudioLoadRequest) req = new AudioLoadRequest(audio_manager, "explosion.wav", true);
AsyncTaskManager::get_global_ptr()->add(req);

// later, once req->done():
PT(AudioSound) sound = (AudioSound *)req->result();
sound->play();
```

## See also

[AudioManager](AudioManager.md), [AsyncTask](../event/AsyncTask.md),
[../pgraph/ModelLoadRequest.md](../pgraph/ModelLoadRequest.md) (closest
sibling pattern), [README.md](README.md)
