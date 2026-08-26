# PStatTimer

**Source:** `panda/src/pstatclient/pStatTimer.{h,I}` (header-only — no `.cxx`)
**Inherits from:** none

A minimal RAII wrapper that starts a [`PStatCollector`](PStatCollector.md) on
construction and stops it on destruction — the standard way to time a scope
of code without manually pairing `start()`/`stop()` calls (and without
leaking a "started" state on an early return or exception). This is the class
referenced by name throughout the engine wherever code times itself against
PStats — e.g. `../cull/README.md`'s note that `CullBin::finish_cull()`/
`draw()` wrap their bodies in a `PStatTimer`.

## Behavior

**Resolves and caches the current thread once, at construction, not per
start/stop call.** The two-argument constructor takes an explicit
`Thread *current_thread`; the one-argument constructor resolves it itself via
`PStatClient::get_global_pstats()->get_current_thread()`. Either way, the
resulting [`PStatThread`](PStatThread.md) is stored in `_thread` and reused
for both the constructor's `start()` and the destructor's `stop()` call —
avoiding a second thread lookup at scope-exit.

**Holds the `PStatCollector` by reference, not by value.** `_collector` is a
`PStatCollector &`, so the caller's collector (typically a `static` local)
must outlive the timer — which it always does in practice, since collectors
are almost always function-static or class-member variables, not stack
locals.

**Compiles to two empty inline constructors and an empty destructor when
`DO_PSTATS` is undefined** — the class exists either way so calling code
never needs `#ifdef`s around its `PStatTimer` usage, matching the pattern of
every other public class in this module.

## API reference

```cpp
class PStatTimer {
public:
  PStatTimer(PStatCollector &collector);
  PStatTimer(PStatCollector &collector, Thread *current_thread);
  ~PStatTimer();
};
```

- Both constructors call `collector.start(_thread)` immediately; the
  destructor calls `collector.stop(_thread)`.
- Not `PUBLISHED` / not exposed to Python — this is a pure C++ RAII
  convenience (Python code times sections via the `pstats-python-profiler`
  integration instead — see [`PStatClient.md`](PStatClient.md)'s API
  reference).

## Usage

```cpp
static PStatCollector cull_pcollector("Cull");

void my_cull_function() {
  PStatTimer timer(cull_pcollector);
  // ... work being timed ...
}  // timer destructor stops the collector here, even on an early return
```

## Related classes

- [`PStatCollector`](PStatCollector.md) — the collector this class
  start()s/stop()s automatically
- [`PStatThread`](PStatThread.md) — resolved once at construction and reused
  at destruction
- `../cull/README.md` — a documented example of `PStatTimer` usage
  (`_cull_this_pcollector`/`_draw_this_pcollector`)
