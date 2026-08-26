# PStatClient

**Source:** `panda/src/pstatclient/pStatClient.{h,I,cxx}`
**Inherits from:** `Thread::PStatsCallback` (see `../pipeline/Thread.md`)

The client-side singleton that owns every collector and thread PStats knows
about and drives the connection to a remote PStats server. Normally there is
exactly one, retrieved via `get_global_pstats()`; constructing your own is
possible ("if extraordinary circumstances require it") but every
[`PStatCollector`](PStatCollector.md) registers itself with whichever client
it's told to use at construction time, so mixing multiple clients needs
deliberate care. If `DO_PSTATS` is undefined, this class compiles to a stub
whose methods are no-ops — see "Core concepts" in the module
[README](README.md).

## Behavior

**Collector 0 is always `"Frame"`, hand-built in the constructor** (not
through `make_collector_with_name()`, since it has no parent) as the implicit
root every unqualified collector name is parented under. The main thread is
registered as thread 0 in the same constructor.

**`make_collector_with_relname()` parses `:`-separated names iteratively**:
it strips leading colons, then repeatedly splits off the first path segment
and resolves/creates a collector for it via `make_collector_with_name()`
before moving to the next segment, so `"Cull:Sort:Detail"` walks through
three levels of parent lookup. `make_collector_with_name()` itself is where
the `"Frame:Frame"` collapse (return the parent unchanged if the requested
child name equals the parent's own name) and the "look up an existing child
by name before creating a new one" dedup both happen — collector creation is
otherwise append-only into a growable array (`_collectors`, doubling from an
initial 128), guarded by `_lock` (a `ReMutex`) for structural writes but read
via `AtomicAdjust::get_ptr()` so lookups don't need to take the lock at all.

**Threads are matched to `Thread` objects by `Thread::get_pstats_index()`,
lazily assigning a new slot on first sight** (`do_make_thread()`). If a
previously-seen thread *name* comes back but the earlier `Thread` object was
deleted (`WPT(Thread) _thread` — a weak pointer — reports `was_deleted()`),
the same `PStatThread` slot is recycled for the new `Thread` rather than
growing the array further, as long as the `sync_name` also matches.
`make_gpu_thread()` is a separate path used only by the GSG to create a
synthetic `"GPU"`-sync-named thread with no backing `Thread` object at all.

**`start()`/`stop()`/`is_started()` operate on a `_nested_count` per
collector-per-thread, not a bool** — see the module README's "Core concepts"
for why. A collector only writes a real `add_start()`/`add_stop()` event into
that thread's `PStatFrameData` on the 0→1 / 1→0 transition; every call is
gated on both `collector->is_active()` (its `PStatCollectorDef` says so —
lazily created via `Collector::get_def()`/`make_def()`) and
`thread->_is_active` (only true once the client has a UDP port from the
server). `set_level()`/`add_level()`/`get_level()` similarly write to a
per-thread `PerThreadData` slot, scaled by the collector def's `_factor`
(set from the `pstats-factor-<name>` config var, see
[`PStatCollectorDef.md`](PStatCollectorDef.md)) each time.

**`disconnect()` resets, but does not destroy, collector/thread state.** It
tears down `_impl` and zeroes every thread's frame number, active flag, and
buffered `PStatFrameData`, and resets every collector's per-thread
`_nested_count` to 0 across every thread — but the collector and thread
*registries* themselves (names, hierarchy, indices) persist, so reconnecting
later doesn't require re-creating any `PStatCollector`/`PStatThread` your
code already holds.

**`main_tick()`/`thread_tick()` are the frame-boundary entry points**, meant
to be called once per frame from application code (or automatically from the
`Panda3D` task manager). `main_tick()` additionally drives the
`DO_MEMORY_USAGE` per-`TypeHandle` reporting described in the module
README, then delegates to `client_main_tick()`, which calls
`PStatClientImpl::client_main_tick()` and `new_frame()` on every thread whose
`sync_name` is `"Main"`. `thread_tick(sync_name)` does the equivalent for any
other named group of threads (used for e.g. an app/cull/draw split where each
stage ticks its own thread group independently).

**Three static callbacks are wired into the global `ClockObject`** at
`get_global_pstats()`'s first-construction time
(`ClockObject::_start_clock_wait`, `_start_clock_busy_wait`,
`_stop_clock_wait`) to time `ClockObject::wait_until()`'s sleep/spin phases
via two dedicated collectors, `"Wait:Clock Wait:Sleep"` and
`"Wait:Clock Wait:Spin"`. The source calls this "a hack around the fact that
we can't let the ClockObject directly create a PStatCollector, because the
pstatclient module depends on putil" (a dependency-direction workaround, not
a design choice `ClockObject` itself could make).

**`activate_hook()`/`deactivate_hook()` implement `Thread::PStatsCallback`**
(see `../pipeline/Thread.md`), invoked by `Thread` on a context switch,
particularly under `SIMPLE_THREADS` where several logical threads share one
OS thread. Each brackets a `"Wait:Thread block"` timing collector by hand,
explicitly avoiding any mutex ("we are called during a context switch, so a
mutex might be dangerous").

## API reference

```cpp
class PStatClient : public Thread::PStatsCallback {
PUBLISHED:
  void set_client_name(const std::string &name);
  std::string get_client_name() const;
  void set_max_rate(double rate);
  double get_max_rate() const;

  int get_num_collectors() const;
  PStatCollector get_collector(int index) const;
  PStatCollectorDef *get_collector_def(int index) const;
  std::string get_collector_name(int index) const;
  std::string get_collector_fullname(int index) const;

  int get_num_threads() const;
  PStatThread get_thread(int index) const;
  std::string get_thread_name(int index) const;
  std::string get_thread_sync_name(int index) const;
  PT(Thread) get_thread_object(int index) const;

  PStatThread get_main_thread() const;
  PStatThread get_current_thread() const;

  double get_real_time() const;

  static bool connect(const std::string &hostname = std::string(), int port = -1);
  static void disconnect();
  static bool is_connected();
  static void resume_after_pause();

  static void main_tick();
  static void thread_tick(const std::string &sync_name);

  static PStatClient *get_global_pstats();
};
```

- `connect()`/`disconnect()`/`is_connected()` are static convenience wrappers
  around `get_global_pstats()->client_connect(...)` etc. — the non-static
  `client_*` methods below them work on any specific `PStatClient` instance.
- `set_max_rate()` caps packets/sec/thread sent to the server (default from
  `pstats-max-rate`).
- `get_collector_fullname()` recursively concatenates parent names with `:`
  up to (but excluding) the root `"Frame"`.
- `resume_after_pause()` re-bases the client's internal clock so a
  deliberately-paused simulation doesn't show as one giant chug frame.
- **Python-only** (`HAVE_PYTHON && DO_PSTATS`, defined in
  `pStatClient_ext.{h,I,cxx}`, not part of the C++ surface above): the
  Python-exposed `connect()`/`disconnect()` wrap the C++ versions and, when
  `pstats-python-profiler` is set, additionally install a `PyEval_SetProfile`
  trace callback that turns every Python function/method call (and C
  function call, via `PyTrace_C_CALL`) into a nested `PStatCollector` under
  `"App:Python"`, named from the call's module/class/function — giving
  per-Python-function timing in the PStats graph without any manual
  instrumentation. The callback is automatically torn down on disconnect or
  if the client is found to have disconnected mid-trace.

## Usage

Not normally constructed directly — call the static `PStatClient::connect()`
(or its Python-exposed equivalent) to connect the global client, then create
[`PStatCollector`](PStatCollector.md)s and time sections of code with
[`PStatTimer`](PStatTimer.md):

```cpp
PStatClient::connect();  // or connect("myhost", 5185)

static PStatCollector my_pcollector("App:MySystem");
{
  PStatTimer timer(my_pcollector);
  // ... code to profile ...
}

PStatClient::main_tick();  // once per frame
```

## Related classes

- [`PStatCollector`](PStatCollector.md) — the handle application code
  actually constructs; delegates to this class's private `start()`/`stop()`/
  `add_level()` methods
- [`PStatClientImpl`](PStatClientImpl.md) — the actual network/protocol
  implementation, created lazily on first use
- [`PStatThread`](PStatThread.md) — handle to one of this client's registered
  threads
- `../pipeline/Thread.md` — `PStatsCallback` base class and
  `get_pstats_index()`/`set_pstats_index()`
