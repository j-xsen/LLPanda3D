# ClockObject

**Source:** `panda/src/putil/clockObject.h` / `.I` / `.cxx` (+ `TimeVal`, same header)
**Inherits:** `ReferenceCount`
**Inherited by:** (none)

Panda's frame clock. Tracks two independent notions of time: **real time**
(`get_real_time()`/`get_long_time()`, wall-clock seconds since construction
or the last `reset()`, always advancing regardless of mode) and **frame
time** (`get_frame_time()`, the discrete time as of the last `tick()` call —
what rendering/animation code should read so everything computed during one
frame agrees on "now"). There is a lazily-constructed global instance
(`get_global_clock()`) that the app framework ticks once per frame; nothing
stops you from creating your own `ClockObject` for an independent local
timer.

## Behavior notes

- **`get_frame_time()`/`get_dt()` read cycled data (`PipelineCycler<CData>`),
  everything else is uncycled.** Frame time, frame count, and dt are stored
  in a `CData` block cycled per pipeline stage/thread so renderer and app
  threads can read a frame-consistent snapshot; `_mode`, `_max_dt`,
  `_degrade_factor`, etc. are plain uncycled members — mutating them
  (`set_mode()`, `set_frame_rate()`, `set_frame_count()`, `tick()`) asserts
  `current_thread->get_pipeline_stage() == 0`, i.e. must happen on the App
  stage, not from a Cull/Draw thread.
- **`Mode` changes what `tick()` computes**, not just a label:
  - `M_normal` — reports real elapsed time each tick; the default.
  - `M_non_real_time` — ignores real time; each tick advances frame time by
    exactly `1/frame_rate` regardless of how long the frame actually took.
    Used for deterministic, non-real-time rendering (e.g. movie capture).
  - `M_limited` — runs as fast as real time allows but never faster than
    `frame_rate`; busy-waits/sleeps at the end of `tick()` if ahead of
    schedule.
  - `M_integer` / `M_integer_limited` — like `M_limited`, but the dt is
    snapped to an integer multiple or divisor of `1/frame_rate` (e.g. if
    frame_rate is 60 and the app is actually running at ~35fps, dt snaps to
    exactly 1/30 rather than reporting jittery real timings).
  - `M_forced` — pretends the app always hits exactly `frame_rate`: slows
    down if running faster, and if running slower, *lies* and reports the
    target dt anyway (real time and reported time diverge).
  - `M_degrade` — real time, but each tick's dt is multiplied by
    `degrade_factor` (>1 slows down, <1 speeds up) via `set_degrade_factor()`
    — a debugging tool to simulate a slower/faster machine.
  - `M_slave` — `tick()` does nothing; the caller must drive `set_frame_time()`
    / `set_frame_count()` manually. Used to slave one clock's frame reporting
    to an external source.
- **`get_dt()` clamps to `max_dt`** (`set_max_dt()`, default -1 = unclamped,
  also settable via the `max-dt` prc variable) only in modes where dt is
  computed from real elapsed time (`M_normal`, `M_limited`, `M_integer*`,
  `M_degrade`); `M_non_real_time`/`M_forced` ignore it since their dt is
  already fixed by `frame_rate`.
- **`set_dt()`/`set_frame_rate()` are two names for the same underlying
  value** (`_user_frame_rate = 1/dt`); which one is meaningful depends on
  mode — irrelevant in `M_normal`, the "target" in `M_limited`/`M_forced`,
  the fixed virtual rate in `M_non_real_time`. `set_dt()` on a non-`M_slave`
  clock asserts `dt != 0`; on `M_slave` any dt including 0 is legal.
- **Average frame rate / max frame duration / deviation are computed from a
  rolling deque of past `tick()` timestamps**, pruned to
  `get_average_frame_rate_interval()` seconds (`set_average_frame_rate_interval(0)`
  disables the tracking entirely and clears the deque — cheap to leave on).
- **`get_global_clock()` lazily constructs via lock-free
  compare-and-exchange** (`AtomicAdjust::compare_and_exchange_ptr`) so
  concurrent first-callers on different threads don't race; the loser's
  redundant instance is simply deleted. The global clock's mode is set from
  the `clock-mode` prc variable at construction.
- **`get_real_time()` vs. `get_long_time()`**: both measure elapsed seconds
  since construction/reset, but from two different OS timers
  (`TrueClock::get_short_time()`/`get_long_time()`). `real_time` is the more
  precise timer for short intervals but can drift over a long session;
  `long_time` is coarser (documented as ~55ms resolution on Windows) but
  doesn't drift — use it for anything that must stay accurate over a long
  run.
- **`reset()`** zeroes real time, frame time, and frame count together;
  `set_real_time()` alone only rebases the real-time origin — frame time
  doesn't change until the next `tick()`.
- **`TimeVal`** (declared in the same header) is an unrelated, small
  POSIX-`timeval`-shaped struct (`tv[2]` = sec, usec) with no behavior of
  its own; not part of `ClockObject`'s API.

## API

### Global access
| Signature | Notes |
|---|---|
| `static ClockObject *get_global_clock()` | Lazily constructs on first call; mode from `clock-mode` prc var |

### Mode
```cpp
enum Mode { M_normal, M_non_real_time, M_forced, M_degrade,
            M_slave, M_limited, M_integer, M_integer_limited };
```
| Signature | Notes |
|---|---|
| `ClockObject(Mode mode = M_normal)` | |
| `void set_mode(Mode)` / `Mode get_mode() const` | Must be called from pipeline stage 0 |

### Frame time / real time
| Signature | Notes |
|---|---|
| `double get_frame_time(Thread* = current) const` | Time as of the last `tick()`; use this for rendering/animation |
| `double get_real_time() const` | Wall-clock seconds since construction/reset; precise, may drift long-term |
| `double get_long_time() const` | Wall-clock seconds since construction/reset; coarser, doesn't drift |
| `void reset()` | Zeroes real time + frame time + frame count together |
| `void set_real_time(double)` | Rebases real-time origin only |
| `void set_frame_time(double, Thread* = current)` | Direct override; normally use `tick()` instead |
| `void tick(Thread* = current)` | Advances frame time/count per the current `Mode`; app-framework calls this once/frame |
| `void sync_frame_time(Thread* = current)` | `M_normal` only: snaps frame time to real time mid-frame without advancing frame count/dt |

### Frame count / dt / rate
| Signature | Notes |
|---|---|
| `int get_frame_count(Thread* = current) const` / `void set_frame_count(int, Thread* = current)` | |
| `double get_net_frame_rate(Thread* = current) const` | `frame_count / frame_time` since reset |
| `double get_dt(Thread* = current) const` | Elapsed time of previous frame, clamped to `max_dt` where applicable |
| `void set_dt(double)` | Sets `1/frame_rate`; asserts nonzero unless `M_slave` |
| `void set_frame_rate(double)` | Same underlying value as `set_dt()`, expressed as Hz |
| `double get_max_dt() const` / `void set_max_dt(double)` | -1 = unclamped (default); also `max-dt` prc var |
| `double get_degrade_factor() const` / `void set_degrade_factor(double)` | Only affects `M_degrade` |

### Frame-rate statistics
| Signature | Notes |
|---|---|
| `void set_average_frame_rate_interval(double)` / `get_average_frame_rate_interval() const` | Window (seconds) for the stats below; 0 disables tracking |
| `double get_average_frame_rate(Thread* = current) const` | Over the tracked interval |
| `double get_max_frame_duration(Thread* = current) const` | Worst single-frame dt over the interval |
| `double calc_frame_rate_deviation(Thread* = current) const` | Standard deviation of frame durations — "chugginess" |

### Misc
| Signature | Notes |
|---|---|
| `bool check_errors(Thread*)` | True if the underlying OS clock reported an error since last call |

## Usage

```cpp
ClockObject *clock = ClockObject::get_global_clock();
clock->set_mode(ClockObject::M_limited);
clock->set_frame_rate(60.0);

// once per frame, typically done by the app framework:
clock->tick();
double dt = clock->get_dt();
double now = clock->get_frame_time();
```

## See also

[AnimInterface.md](AnimInterface.md) (play/pose timing built on
`get_frame_time()`) · [README.md](README.md)
