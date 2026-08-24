# AnimInterface

**Source:** `panda/src/putil/animInterface.h` / `.I` / `.cxx`
**Inherits:** (none)
**Inherited by:** `AnimControl` (see [chan/AnimControl.md](../chan/AnimControl.md)), and other frame-based animatable classes

Base class providing the shared play/loop/pingpong/pose control-flow used by
anything that plays back a fixed-length, frame-indexed animation. It owns no
animation data itself — a subclass calls the protected `set_num_frames()`
and `set_frame_rate()` to describe the animation, and overrides
`animation_activated()` if it needs to react when playback starts. All
current-frame math is time-based: nothing is polled or ticked — the current
frame is always computed on demand from
`ClockObject::get_global_clock()->get_frame_time()` and the play state
recorded at the last `play()`/`loop()`/`pingpong()`/`pose()` call.

## Behavior notes

- **Current frame is derived, not stored.** Calling `play()` etc. just
  records `_play_mode`, `_start_time` (= current global-clock frame time),
  and the from/to range; every subsequent `get_frame()`/`get_full_fframe()`
  call recomputes the position from `(now - _start_time) * effective_frame_rate`.
  There's no per-frame update method to call — the "animation" advances
  purely because time passes between calls.
- **`from`/`to` frame numbers may exceed `get_num_frames()`.**
  `play(0, get_num_frames() * 2)` plays the animation through twice then
  stops; the class doesn't validate the range against the frame count.
- **`PM_play` clamps; `PM_loop`/`PM_pingpong` wrap.** In play mode, elapsed
  time beyond the range is clamped to the end frame (`is_playing()` becomes
  false once past it). In loop mode, elapsed time is taken modulo the range
  length; in pingpong mode it's modulo `2×` the range length, mirrored back
  for the second half — so `get_full_fframe()` can return values past `to`
  only in the pingpong return leg.
  - If `_effective_frame_rate < 0` (negative `play_rate`), `PM_play` starts
    positioned at the *end* of the range and plays backward; `is_playing()`
    checks `get_f() > 0` instead of `< _play_frames` in that case.
- **Restarting with `restart=false` preserves apparent position.** `loop()`
  / `pingpong()` called with `restart=false` compute the current fractional
  frame *before* changing state, clamp it into the new `[from, to]` range,
  and back-date `_start_time` (or, if currently paused, adjust `_paused_f`)
  so playback continues from that position rather than jumping to `from`.
- **`play_rate` and `frame_rate` are independent knobs that combine
  multiplicatively** into `_effective_frame_rate`. `frame_rate` is the
  animation's native rate (set once by the subclass via the protected
  `set_frame_rate()`); `play_rate` is the public playback-speed multiplier
  (`set_play_rate()` — 1.0 normal, 2.0 double speed, 0.0 pauses, negative
  reverses). Changing either mid-playback (`internal_set_rate()`)
  recalculates `_start_time` so the current apparent frame doesn't jump.
- **Setting `play_rate` to exactly 0.0 pauses** by recording the current
  frame into `_paused_f`/`_paused = true`; unpausing (nonzero rate again)
  recomputes `_start_time` from the frozen frame so playback resumes exactly
  where it left off rather than snapping forward by the elapsed real time.
- **`stop()` poses at the current frame and deliberately does not call
  `animation_activated()`** — every other transition (`play`, `loop`,
  `pingpong`, `pose`) does. The comment in the source is explicit: stopping
  should not be treated as "activating" the animation.
- **`get_num_frames()` is virtual** — a subclass may override it to return a
  value that changes at runtime (e.g. depending on which sub-animation is
  bound), rather than relying on the fixed `_num_frames` set via
  `set_num_frames()`.
- **Bam serialization is provided (`write_datagram`/`fillin`) but protected**
  — this class exists to be embedded as part of a larger writable object
  (like `AnimControl`), not written to a bam file on its own.

## API

```cpp
enum PlayMode { PM_pose, PM_play, PM_loop, PM_pingpong };  // private, not exposed
```

### Playback control
| Signature | Notes |
|---|---|
| `void play()` | Full range, once |
| `void play(double from, double to)` | Inclusive range, stops at `to` |
| `void loop(bool restart)` | Full range, indefinitely |
| `void loop(bool restart, double from, double to)` | `restart=false` continues from current position |
| `void pingpong(bool restart)` / `pingpong(bool restart, double from, double to)` | Bounces between `from` and `to` |
| `void stop()` | Freezes at current frame; does NOT call `animation_activated()` |
| `void pose(double frame)` | Jumps to and holds a specific frame |

### Rate / position
| Signature | Notes |
|---|---|
| `void set_play_rate(double)` / `double get_play_rate() const` | 1.0 normal; 0.0 pauses; negative reverses |
| `double get_frame_rate() const` | Native rate, set by the subclass |
| `virtual int get_num_frames() const` | Virtual — subclass may override |
| `int get_frame() const` | Current frame, wrapped to `[0, num_frames)` |
| `int get_next_frame() const` | `get_frame() + 1`, wrapped (clamped in `PM_play` near the end) |
| `double get_frac() const` | Fractional part; `get_full_frame() + get_frac() == get_full_fframe()` |
| `int get_full_frame() const` / `double get_full_fframe() const` | Unwrapped — may exceed `num_frames` for out-of-range `to`; `fframe` may hit `to+1` exactly at natural end where `full_frame` will not |
| `bool is_playing() const` | False once a `PM_play` run reaches its end, or after `stop()`/`pose()` |

### For subclasses (protected)
| Signature | Notes |
|---|---|
| `void set_frame_rate(double)` | Declares the animation's native rate |
| `void set_num_frames(int)` | Declares the frame count (unless `get_num_frames()` is overridden) |
| `virtual void animation_activated()` | Hook called after `play`/`loop`/`pingpong`/`pose` (not `stop`) |

## Usage

```cpp
class MyAnim : public AnimInterface {
public:
  MyAnim() { set_frame_rate(24.0); set_num_frames(100); }
};

MyAnim anim;
anim.loop(true);              // start looping from frame 0
anim.set_play_rate(0.5);      // half speed, position preserved
int f = anim.get_frame();     // 0 <= f < 100, derived from elapsed time
```

## See also

[ClockObject.md](ClockObject.md) (the time source every frame computation
reads) · [chan/AnimControl.md](../chan/AnimControl.md) (concrete subclass) ·
[README.md](README.md)
