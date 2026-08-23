# NullAudioManager

**Source:** `panda/src/audio/nullAudioManager.h` / `.cxx`
**Inherits:** [AudioManager](AudioManager.md)

The automatic no-op fallback manager. Selected by
[AudioManager](AudioManager.md)`::create_AudioManager()` whenever no real
backend is available: `audio-library-name` is `"null"`/empty (the default),
the requested backend's dlopen fails, or a loaded backend reports
`!is_valid()`. Its own header points elsewhere for implementers: "If you're
looking for a starting place for a new AudioManager, please consider looking
at the milesAudioManager" (outside this module).

## Behavior notes

- **`is_valid()` always returns `false`** — this is the flag
  `create_AudioManager()` checks (via an exact-type guard) to avoid an
  infinite fallback loop when *this* class is itself the result.
- **Both `get_sound()` overloads** (filename or `MovieAudio*`) return
  `AudioManager::get_null_sound()` — the shared singleton — regardless of
  what filename or source is passed, so no actual load ever happens.
- **Constructor logs via `audio_info()`**, so selecting this manager is
  visible in Notify output (`audio` category, info level) even though
  nothing else about it does anything.
- Every other virtual (`uncache_sound`, `clear_cache`, `set_cache_limit`,
  volume/active/concurrency getters and setters, `reduce_sounds_playing_to`,
  `stop_all_sounds`, all `audio_3d_*` attribute/factor methods) is a stub:
  setters no-op, getters return `0`/`false`.

## API

All overrides are stubs; only the return-value behavior differs from a
generic no-op:

| Method | Stub behavior |
|---|---|
| `is_valid()` | Always `false` |
| `get_sound(Filename\|MovieAudio*, ...)` | Always returns `get_null_sound()` |
| Everything else (`set_*`/`get_*`/`uncache_sound`/`clear_cache`/`stop_all_sounds`/`reduce_sounds_playing_to`/`audio_3d_*`) | No-op setters; getters return `0`/`false` |

## See also

[AudioManager](AudioManager.md), [NullAudioSound](NullAudioSound.md), [README.md](README.md)
