# AudioSound

**Source:** `panda/src/audio/audioSound.h` / `.I` / `.cxx`
**Inherits:** `TypedReferenceCount`

A single loaded/streaming sound. Obtained only from
[AudioManager](AudioManager.md)`::get_sound()` — never constructed directly
(the constructor is `protected` with `friend class AudioManager`). Most 3D
and speaker-specific methods are non-pure no-op stubs here; a real backend
overrides only the subset it supports.

## Behavior notes

- **Set loop/volume/balance before calling `play()`.** The header's own
  compatibility note: setting these while a sound is already playing is
  implementation-specific and may not take effect. Calling `play()` a second
  time before the sound finishes restarts it (a skip/stutter effect), rather
  than layering a second instance.
- **`SoundStatus`** is `BAD`/`READY`/`PLAYING` with a matching `operator<<`
  for logging.
- **`configure_filters()`** default implementation mirrors
  `AudioManager::configure_filters()` exactly: returns `true` only for an
  empty `FilterProperties` chain, `false` otherwise. A real backend
  overrides this to actually apply per-sound filters.
- **Speaker methods split by backend, explicitly per `.cxx` comments:**
  `get_speaker_mix()`/`set_speaker_mix()` are "for use only with FMOD";
  `get_speaker_level()`/`set_speaker_levels()` are "for use only with
  Miles." The header notes both exist because the two APIs' speaker models
  don't map cleanly onto each other, with a "hopefully reconciled someday"
  aspiration that never happened.
- **`get_time()` semantics:** doesn't advance while the sound isn't playing
  (e.g. paused via `stop()`), and on each loop iteration after the first,
  restarts from the beginning of the sound rather than continuing to
  advance monotonically — checking `get_time() / length()` for percent
  complete only makes sense within a single loop iteration.
- **`get_priority()`/`set_priority()`** are no-op stubs in the base class
  (return `0`); meaningful only where backend-implemented.
- **No `set_name()`** — intentional, per an explicit header comment; a
  sound's name is fixed at load time.

## API

### Playback control
| Signature | Notes |
|---|---|
| `virtual void play() = 0` | Restarts if already playing |
| `virtual void stop() = 0` | |
| `virtual SoundStatus status() const = 0` | `BAD`/`READY`/`PLAYING` |

### Loop / time / volume / balance / rate
| Signature | Notes |
|---|---|
| `virtual void set_loop(bool=true) / get_loop() const = 0` | Inits `false` |
| `virtual void set_loop_count(unsigned long=1) / get_loop_count() const = 0` | `0`=forever, `1`=once, `n`=n times |
| `virtual void set_time(PN_stdfloat=0.0) / get_time() const = 0` | Seek position in seconds; see Behavior notes |
| `virtual void set_volume(PN_stdfloat=1.0) / get_volume() const = 0` | `0`=min, `1.0`=max |
| `virtual void set_balance(PN_stdfloat=0.0) / get_balance() const = 0` | `-1.0`=hard left, `1.0`=hard right |
| `virtual void set_play_rate(PN_stdfloat=1.0) / get_play_rate() const = 0` | Any positive value |
| `virtual PN_stdfloat length() const = 0` | Total playing time in seconds |

### 3D attributes
| Signature | Notes |
|---|---|
| `virtual void set_3d_attributes(px,py,pz, vx,vy,vz) / get_3d_attributes(...)` | Emitter position + velocity (units/sec); no-op in base |
| `virtual void set_3d_min_distance(PN_stdfloat) / get_3d_min_distance() const` | Default `1.0`; `<1.0` farther/slower falloff, `>1.0` closer/faster |
| `virtual void set_3d_max_distance(PN_stdfloat) / get_3d_max_distance() const` | Default `1000000000.0`; sound doesn't stop here, just stops getting quieter |

### Speaker mix (FMOD) / levels (Miles)
| Signature | Notes |
|---|---|
| `virtual PN_stdfloat get_speaker_mix(int speaker) / void set_speaker_mix(frontleft, frontright, center, sub, backleft, backright, sideleft, sideright)` | FMOD only |
| `virtual PN_stdfloat get_speaker_level(int index) / void set_speaker_levels(level1..level9)` | Miles only |

### Priority / filters
| Signature | Notes |
|---|---|
| `virtual int get_priority() / void set_priority(int)` | No-op in base |
| `virtual bool configure_filters(FilterProperties*)` | Per-sound DSP chain; base only accepts an empty chain |

### Status / output / events
| Signature | Notes |
|---|---|
| `virtual void set_finished_event(const std::string&) / get_finished_event() const = 0` | Event thrown when playback finishes; pass `""` to clear |
| `virtual const std::string &get_name() const = 0` | No corresponding setter |
| `virtual void set_active(bool=true) / get_active() const = 0` | Inits to the owning manager's active state |
| `virtual void output(std::ostream&) const / write(std::ostream&) const` | |

## Usage

```cpp
PT(AudioSound) sound = audio_manager->get_sound("music/theme.ogg");
sound->set_loop(true);
sound->set_volume(0.6f);
sound->set_finished_event("theme-done");  // no-op if looping forever
sound->play();

if (sound->status() == AudioSound::PLAYING) {
  PN_stdfloat percent = sound->get_time() / sound->length();
}
```

## See also

[AudioManager](AudioManager.md), [NullAudioSound](NullAudioSound.md),
[FilterProperties](FilterProperties.md),
[../pgraph/AudioVolumeAttrib.md](../pgraph/AudioVolumeAttrib.md) (a different,
scene-graph-driven volume mechanism), [README.md](README.md)
