# AudioManager

**Source:** `panda/src/audio/audioManager.h` / `.I` / `.cxx`
**Inherits:** `TypedReferenceCount`

Owns a pool of sounds for one category (sound effects, music, etc. — the
convention is one `AudioManager` per category). Almost entirely abstract:
the base class implements only the factory/fallback machinery, the shared
null-sound singleton, and no-op stubs for the 3D/speaker/filter methods a
minimal backend doesn't need to support.

## Behavior notes

- **Factory pattern (`create_AudioManager()`):** checks
  `_create_AudioManager` (set via `register_AudioManager_creator()`) first —
  this lets a statically-linked backend register itself without dlopen.
  Otherwise, on first call only (`static bool lib_load` one-shot guard), it
  dlopens `lib<audio-library-name>.so` from the plugin path, strips a
  leading `p3` from the library name, and looks up
  `get_audio_manager_func_<name>`. Whatever manager results, if it isn't
  exactly `NullAudioManager` and reports `!is_valid()`, it's discarded in
  favor of a fresh `NullAudioManager` — this check runs both on the
  registered-creator path and the dlopen path, so a broken backend can never
  make it back to calling code.
- **`register_AudioManager_creator()`** asserts (`nassertv`) that it's either
  the first registration or an exact repeat of the same function pointer —
  registering two different creators is a program error, not something it
  silently resolves.
- **`get_null_sound()`** lazily builds one `NullAudioSound`, `ref()`s it, and
  installs it via `AtomicAdjust::compare_and_exchange_ptr` on `_null_sound`;
  if another thread won the race, the loser's fresh sound is
  `unref_delete()`d. Thread-safe without a mutex; happens at most twice per
  manager even under contention.
- **"Avoid adding data members to this mostly abstract base class"** — an
  explicit design comment in the header, so `.p` protected members should be
  expected to stay minimal.
- **`SpeakerModeCategory`** enumerants line up one-to-one with FMOD's
  `SPEAKERMODE` enum by design.
- **`StreamMode`** (`SM_heuristic`/`SM_sample`/`SM_stream`, passed to
  `get_sound()`) ties into the `audio-preload-threshold` config var in real
  backends — `SM_heuristic` lets the backend decide based on file size.
- **Most 3D/doppler/distance-factor virtuals are non-pure no-ops** in this
  base class (`audio_3d_set_listener_attributes`, `audio_3d_*_distance_factor`,
  `audio_3d_*_doppler_factor`, `audio_3d_*_drop_off_factor`,
  `get_speaker_setup`/`set_speaker_setup`) — only OpenAL/FMOD backends
  override them with real behavior. A minimal backend can ignore spatial
  audio entirely and still compile.
- **`set_speaker_configuration()` is explicitly Miles-only** per its `.cxx`
  comment; other backends leave it a no-op.
- **`get_dls_pathname()`** branches by platform when `audio-dls-file` isn't
  set: reads the Windows Registry (`SOFTWARE\Microsoft\DirectMusic`,
  `GMFilePath`) falling back to `<system dir>/drivers/gm.dls` on Windows;
  returns a hardcoded OSX 10.4-era CoreAudio path on OSX; returns an empty
  `Filename` on any other platform.
- **`shutdown()` and `update()`** are both no-ops in the base class but are
  meaningful hooks: `shutdown()` is documented to invalidate all
  managers/sounds system-wide (recreate everything to play sound again after
  calling it); `update()` must be called every frame or a real backend may
  develop buffering/mixing problems.

## API

### Construction / factory
| Signature | Notes |
|---|---|
| `static PT(AudioManager) create_AudioManager()` | The only way to get a manager; see Behavior notes |
| `static void register_AudioManager_creator(Create_AudioManager_proc*)` | For statically-linked backends |
| `virtual void shutdown()` | Invalidates all managers/sounds system-wide |
| `virtual bool is_valid() = 0` | Not required to check before use — invalid managers are safe to call, just silent |

### Sound retrieval & caching
| Signature | Notes |
|---|---|
| `virtual PT(AudioSound) get_sound(const Filename&, bool positional=false, int mode=SM_heuristic) = 0` | |
| `virtual PT(AudioSound) get_sound(MovieAudio*, bool positional=false, int mode=SM_heuristic) = 0` | |
| `PT(AudioSound) get_null_sound()` | Shared no-op sound singleton |
| `virtual void uncache_sound(const Filename&) = 0` | Doesn't affect already-obtained `AudioSound`s |
| `virtual void clear_cache() = 0` | |
| `virtual void set_cache_limit(unsigned int) / get_cache_limit() const = 0` | |

### Volume / active / concurrency
| Signature | Notes |
|---|---|
| `virtual void set_volume(PN_stdfloat) / get_volume() const = 0` | `0`=min, `1.0`=max, inits to `1.0` |
| `virtual void set_active(bool) / get_active() const = 0` | Deactivating stops playing sounds; reactivating restarts looping sounds from the beginning |
| `virtual void set_concurrent_sound_limit(unsigned int limit=0) / get_concurrent_sound_limit() const = 0` | `0`=unlimited, `1`=mutually exclusive |
| `virtual void reduce_sounds_playing_to(unsigned int count) = 0` | |
| `virtual void stop_all_sounds() = 0` | Equivalent to `reduce_sounds_playing_to(0)`, may be more efficient |
| `virtual void update()` | Call every frame |

### 3D listener attributes
| Signature | Notes |
|---|---|
| `virtual void audio_3d_set_listener_attributes(px,py,pz, vx,vy,vz, fx,fy,fz, ux,uy,uz)` | Position, velocity (units/sec), forward vector, up vector |
| `virtual void audio_3d_get_listener_attributes(...)` | |
| `virtual void audio_3d_set_distance_factor(PN_stdfloat) / audio_3d_get_distance_factor() const` | Units-per-meter; OpenAL/FMOD only |
| `virtual void audio_3d_set_doppler_factor(PN_stdfloat) / audio_3d_get_doppler_factor() const` | `>1.0` exaggerated, `<1.0` diminished |
| `virtual void audio_3d_set_drop_off_factor(PN_stdfloat) / audio_3d_get_drop_off_factor() const` | `>1.0` faster falloff, `<1.0` slower |

### Speaker / filter config
| Signature | Notes |
|---|---|
| `virtual int get_speaker_setup() / void set_speaker_setup(SpeakerModeCategory)` | |
| `virtual void set_speaker_configuration(LVecBase3 *speaker1..speaker9)` | Miles only |
| `virtual bool configure_filters(FilterProperties*)` | Global DSP chain; base only accepts an empty chain |
| `static Filename get_dls_pathname()` | See Behavior notes for platform fallback |

### Output / write
| Signature | Notes |
|---|---|
| `virtual void output(std::ostream&) const` | |
| `virtual void write(std::ostream&) const` | |

## Usage

```cpp
// One AudioManager per category, per the header's own recommendation:
PT(AudioManager) sfx_manager = AudioManager::create_AudioManager();
PT(AudioManager) music_manager = AudioManager::create_AudioManager();

PT(AudioSound) sfx = sfx_manager->get_sound("neatSfx.mp3");
sfx->set_loop(false);
sfx->set_volume(0.8f);
sfx->play();

// Every frame:
sfx_manager->update();
```

## See also

[AudioSound](AudioSound.md), [NullAudioManager](NullAudioManager.md),
[FilterProperties](FilterProperties.md), [AudioLoadRequest](AudioLoadRequest.md),
[README.md](README.md)
