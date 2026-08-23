# Audio — Panda3D's Abstract Audio API

**Source:** `panda/src/audio/` · **Library:** `libp3audio` · **Notify category:** `audio`

A pure abstraction layer over sound playback. `AudioManager` and `AudioSound`
are near-fully-abstract base classes — nearly every method that touches an
actual sound device is either pure virtual or a documented no-op stub in this
module. The real work happens in backend implementations that live outside
this directory (Miles/OpenAL/FMOD in `panda/src/audiotraits/`, not covered
here) and are selected at runtime by name, not compiled against directly.
When no backend is available (or none was requested), `NullAudioManager` and
`NullAudioSound` stand in automatically as silent no-ops — application code
never needs to check whether a real audio system loaded.

This module is unrelated to [`AudioVolumeAttrib`](../pgraph/AudioVolumeAttrib.md)
in pgraph: that's a `RenderAttrib` that scales positional-audio volume based
on scene graph position, a scene-graph-driven mechanism layered on top of
whatever `AudioSound` is actually playing. It's cross-linked below, not
duplicated here.

## Class map

```
TypedReferenceCount
├── AudioManager            (AudioManager.md)
│   └── NullAudioManager    (NullAudioManager.md)
├── AudioSound              (AudioSound.md)
│   └── NullAudioSound      (NullAudioSound.md)
└── FilterProperties        (FilterProperties.md)   — standalone value/config type

AsyncTask (../event/AsyncTask.md)
└── AudioLoadRequest        (AudioLoadRequest.md)
```

## Core concepts

**Getting a manager: factory + automatic fallback.** `AudioManager::create_AudioManager()`
is the only way to get a manager. It first checks for a statically-registered
creator function (`register_AudioManager_creator()` — used when a backend is
linked in directly); failing that, it `dlopen`s `lib<audio-library-name>.so`
from the plugin path (config var `audio-library-name`, default `"null"`) and
looks up a `get_audio_manager_func_<name>` symbol to obtain the real factory.
If the dlopen fails, the symbol is missing, or the resulting manager reports
`is_valid() == false`, it falls back to `NullAudioManager` — guarded against
infinite recursion by checking `is_exact_type(NullAudioManager::get_class_type())`
first. In short: asking for audio never fails outright, it just silently
degrades to no sound.

**Sounds are obtained, not constructed.** `AudioSound`'s constructor is
`protected` with `friend class AudioManager` — the only way to get one is
`AudioManager::get_sound()`. It's a plain ref-counted handle (`TypedReferenceCount`,
not a `PandaNode`), so it has no place in the scene graph on its own.

**The shared null-sound singleton.** `AudioManager::get_null_sound()` lazily
creates one `NullAudioSound` per manager instance using an atomic
compare-and-exchange on `AtomicAdjust::Pointer` (thread-safe without a
mutex), and hands back the same instance on every subsequent call. Both
`NullAudioManager::get_sound()` and any real backend can return this same
singleton to represent "sound object exists but plays nothing."

**Filter chains are generic and append-only.** `FilterProperties` holds an
ordered `pvector<FilterConfig>` — each entry is a `FilterType` tag plus 14
generic float slots whose meaning depends on the filter. There's no per-entry
edit or remove, only `clear()` and re-adding. The same object is accepted
identically by `AudioManager::configure_filters()` (the global DSP chain) and
`AudioSound::configure_filters()` (a per-sound chain) — both base
implementations only claim to support an *empty* chain (return `true` for
`clear()`ed properties, `false` otherwise); a real backend overrides these to
actually apply filters and report what it supports.

**`AudioLoadRequest` isn't tied to `Loader`.** It's a thin `AsyncTask`
subclass that calls `AudioManager::get_sound()` synchronously inside
`do_task()`. Despite the doc comment mentioning `Loader` (pgraph), it works
with *any* `AsyncTaskManager` — construct it, add it to a task manager, and
read the result.

**`AudioManager::update()` must be called every frame.** The base
implementation is a no-op, but real backends rely on it for buffering/mixing
housekeeping; skipping it can cause playback problems.

## Config variables (`config_audio.h`)

| Variable | Default | Meaning |
|---|---|---|
| `audio-active` (Bool) | `true` | Global on/off for the audio system. |
| `audio-cache-limit` (Int) | `15` | Number of sounds kept in an `AudioManager`'s pool cache. |
| `audio-volume` (Double) | `1.0` | Global volume scale. |
| `audio-library-name` (String) | `"null"` | Backend to dlopen (e.g. `"p3openal_audio"`, `"p3fmod_audio"`). `"null"` (or empty) skips loading and uses `NullAudioManager` directly. |
| `audio-dls-file` (Filename) | `""` | DLS instrument-set file for MIDI playback; falls back to a per-OS default (Windows Registry lookup, hardcoded OSX CoreAudio path) via `AudioManager::get_dls_pathname()` if unset. |
| `fmod-number-of-sound-channels` (Int) | `128` | Max concurrent FMOD channels. |
| `fmod-speaker-mode` (Enum `FmodSpeakerMode`) | `FSM_unspecified` | Speaker layout (`raw`/`mono`/`stereo`/`quad`/`surround`/`5.1`/`7.1`); values line up 1:1 with FMOD's `SPEAKERMODE` enum. Supersedes... |
| `fmod-use-surround-sound` (Bool, **deprecated**) | `false` | Superseded by `fmod-speaker-mode`; kept only for backward compatibility. |
| `audio-doppler-factor` (Double) | `1.0` | OpenAL/FMOD only. `>1.0` exaggerates Doppler shift, `<1.0` diminishes it. |
| `audio-distance-factor` (Double) | `1.0` | OpenAL/FMOD only. Units-per-meter for 3D audio distance math. |
| `audio-drop-off-factor` (Double) | `1.0` | OpenAL/FMOD only. `>1.0` faster volume falloff with distance, `<1.0` slower. |
| `audio-buffering-seconds` (Double) | `3.0` | Streaming-audio buffer size in seconds; too-small values under load cause stutter. |
| `audio-preload-threshold` (Int) | `1000000` | Decompressed-size cutoff (bytes) above which a sound streams from disk instead of loading fully into RAM. |
| `audio-software-midi` (Bool) | `true` | Miles only. |
| `audio-play-midi` / `audio-play-wave` / `audio-play-mp3` (Bool) | `true` | Miles only; per-format playback toggles. |
| `audio-output-rate` (Int) | `22050` | Miles only. |
| `audio-output-bits` (Int) | `16` | Miles only. |
| `audio-output-channels` (Int) | `2` | Miles only. |

`audio-min-hw-channels` is defined in `config_audio.cxx` but never declared
`extern` in the header — it's unreachable outside that translation unit and
effectively dead.

## File index

| File | Documents |
|---|---|
| `audioManager.h/.I/.cxx` | [AudioManager](AudioManager.md) |
| `audioSound.h/.I/.cxx` | [AudioSound](AudioSound.md) |
| `audioLoadRequest.h/.I/.cxx` | [AudioLoadRequest](AudioLoadRequest.md) |
| `filterProperties.h/.I/.cxx` | [FilterProperties](FilterProperties.md) |
| `nullAudioManager.h/.cxx` | [NullAudioManager](NullAudioManager.md) |
| `nullAudioSound.h/.cxx` | [NullAudioSound](NullAudioSound.md) |
| `config_audio.h/.cxx` | Config vars, folded into this README above |
| `audio.h` | Pure aggregator header (`#include`s the four class headers); not documented separately |

## Status

Done (2026-08-23). Backend implementations (`panda/src/audiotraits/` —
Miles/OpenAL/FMOD) and media decoding (`panda/src/movies/`) are not covered
by this pass; see the root [README.md](../../README.md) status table.

## See also

- [../pgraph/AudioVolumeAttrib.md](../pgraph/AudioVolumeAttrib.md) — scene-graph-driven volume scale, a different mechanism from anything here.
- [../pgraph/ModelLoadRequest.md](../pgraph/ModelLoadRequest.md) — the closest sibling pattern to `AudioLoadRequest` (an `AsyncTask` wrapping an async load).
- [../event/AsyncTask.md](../event/AsyncTask.md) / [AsyncTaskManager](../event/README.md) — what `AudioLoadRequest` runs on.
