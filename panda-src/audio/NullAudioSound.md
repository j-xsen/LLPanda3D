# NullAudioSound

**Source:** `panda/src/audio/nullAudioSound.h` / `.cxx`
**Inherits:** [AudioSound](AudioSound.md)

"This class intentionally does next to nothing. It's used as a placeholder
when you don't want a sound system." (verbatim header comment.) In practice
obtained via
[AudioManager](AudioManager.md)`::get_null_sound()` — a shared singleton per
manager — even though nothing prevents constructing it directly.

## Behavior notes

- **`status()` always returns `READY`, not `BAD`**, despite doing nothing —
  callers that check status to detect a failed/invalid sound won't catch a
  null sound this way; it presents as a normal, playable, silent sound.
- All getters return `0`/`false`/a shared static empty string; all setters
  and `play()`/`stop()` are no-ops.
- The source contains a genuine authored gotcha as a comment directly above
  the constructor: `// why protect the constructor?!? protected:` — the
  constructor is in fact **public** (declared under the class's `public:`
  section, not `protected:`), despite the comment suggesting it was meant to
  be construction-restricted like `AudioSound`'s own protected constructor.
  `friend class NullAudioManager` is declared but has no effect given the
  public constructor.
- Its own header likewise points at `milesAudioManager` (outside this
  module) as the reference starting point for implementing a real backend.

## API

All overrides are stubs, matching [AudioSound](AudioSound.md)'s full
interface:

| Method | Stub behavior |
|---|---|
| `status()` | Always `READY` (not `BAD`) |
| `get_loop()`, `get_active()` | Always `false` |
| `get_loop_count()`, `get_time()`, `get_volume()`, `get_balance()`, `get_play_rate()`, `length()`, `get_3d_min_distance()`, `get_3d_max_distance()` | Always `0` |
| `get_finished_event()`, `get_name()` | Always a shared empty `std::string` |
| `play()`, `stop()`, all `set_*`, `set_3d_attributes()`, `get_3d_attributes()` | No-ops |

## See also

[AudioSound](AudioSound.md), [NullAudioManager](NullAudioManager.md), [README.md](README.md)
