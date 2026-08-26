# PStatThread

**Source:** `panda/src/pstatclient/pStatThread.{h,I,cxx}`
**Inherits from:** none

A lightweight handle representing one thread of execution to PStats,
corresponding 1:1 with a Panda `Thread` instance. Each collector maintains
independent timing/level state per `PStatThread`, so two threads running the
same collector don't interfere with each other's start/stop counts (see the
module [README](README.md)'s "Core concepts").

## Behavior

**The `(Thread*, PStatClient*)` constructor is the normal entry point, and
resolves lazily through `Thread::get_pstats_index()`.** If the given
`Thread` already has a PStats index assigned, this just wraps that index; if
not (`thread_index == -1`), it calls `client->make_thread(thread)` to
register the thread for the first time — which in turn calls
`Thread::set_pstats_index()`/`set_pstats_callback()` on the `Thread`, wiring
it up to receive `activate_hook()`/`deactivate_hook()` calls (see
`../pipeline/Thread.md`). Without `DO_PSTATS` defined, this constructor
short-circuits to an invalid `(nullptr, 0)` handle instead.

**`new_frame()`/`add_frame()` are thin forwards into
[`PStatClientImpl`](PStatClientImpl.md).** `new_frame()` must be called at
the start of every "frame" (whatever that means for the calling code) to
flush the accumulated stats for this thread and start a fresh one;
`PStatClient::thread_tick(sync_name)` calls this automatically for every
thread sharing that sync name. `add_frame()` is a lower-level variant that
takes an already-assembled [`PStatFrameData`](PStatFrameData.md) instead of
relying on the client's own accumulated buffer.

**`get_thread()` returns different things depending on `DO_PSTATS`**: with
stats compiled in, it resolves back through
`PStatClient::get_thread_object()` (which may return `nullptr` if the
underlying `Thread` was since deleted, since it's held via `WPT(Thread)` — a
weak pointer); without stats compiled in, it always returns
`Thread::get_current_thread()` regardless of which `PStatThread` this is —
a stub-mode approximation, not a faithful lookup.

## API reference

```cpp
class PStatThread {
PUBLISHED:
  PStatThread(PStatClient *client, int index);
  PStatThread(Thread *thread, PStatClient *client = nullptr);

  PStatThread(const PStatThread &copy);
  void operator = (const PStatThread &copy);

  void new_frame();
  void add_frame(const PStatFrameData &frame_data);

  Thread *get_thread() const;
  int get_index() const;
};
```

- The default constructor (public, not `PUBLISHED`) leaves the handle
  uninitialized — normally only used internally before an assignment.
- `get_index()` returns the raw thread index within the owning
  `PStatClient`, used by [`PStatCollector`](PStatCollector.md)'s
  thread-explicit overloads.

## Usage

Usually obtained via `PStatClient::get_current_thread()` or
`get_main_thread()` rather than constructed directly, then passed to a
[`PStatCollector`](PStatCollector.md)'s thread-explicit
`start(thread)`/`stop(thread)`/`add_level(thread, ...)` overloads to attribute
work to a specific thread other than the caller's own:

```cpp
PStatThread gpu_thread = gsg->get_pstats_thread();  // e.g. a GPU-sync thread
draw_pcollector.start(gpu_thread);
// ...
draw_pcollector.stop(gpu_thread);
```

## Related classes

- [`PStatCollector`](PStatCollector.md) — thread-explicit overloads take a
  `const PStatThread &`
- [`PStatClient`](PStatClient.md) — owns the actual per-thread state this
  handle indexes into
- [`PStatTimer`](PStatTimer.md) — resolves the current thread via
  `PStatClient::get_current_thread()` internally
- `../pipeline/Thread.md` — the underlying `Thread` class and its
  `PStatsCallback` hook
