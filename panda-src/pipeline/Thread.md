# Thread

**Source:** `panda/src/pipeline/thread.{h,I,cxx}`
**Inherits from:** `TypedReferenceCount`, `Namable`

A thread; that is, a lightweight process. This is an abstract base class — to
use it, subclass it and override `thread_main()`. The thread itself keeps a
reference count on the `Thread` object while it is running, so a
fire-and-forget thread (`start(priority, /* joinable */ false)`, never stored)
automatically destructs itself when `thread_main()` returns and nothing else
references it.

## Behavior

`start()` refuses to run if the `support-threads` config variable is false,
and otherwise delegates to `ThreadImpl::start()` (see Implementation
variants). `join()` blocks until the thread terminates, or returns
immediately if it already has.

Every `Thread` carries a **pipeline stage** number (`get_pipeline_stage()`,
default 0), which selects which stage of pipelined data (see
[`Pipeline`](Pipeline.md), [`PipelineCycler`](PipelineCycler.md)) this thread
observes. An application thread typically leaves it at 0; a render thread may
set it to 1 or 2 to operate on the previous (or second-previous) frame's data.
`set_pipeline_stage()` is a no-op that warns if called with a nonzero value
when `THREADED_PIPELINE` isn't compiled in.

`get_current_thread()` always returns a valid, non-null pointer, even for
threads spawned entirely outside Panda: such a thread gets back the single
shared [`ExternalThread`](ExternalThread.md) object, unless it was explicitly
associated with its own `Thread` object via `bind_thread()`. `bind_thread()`
is how you give an externally-created thread (e.g. a thread Python spawned
outside `PythonThread`) its own identity for PStats purposes — without it,
every external thread shares one `ExternalThread` and shows up as a single
line in the PStats graph.

`Thread::sleep()`, `force_yield()`, and `consider_yield()` all forward
directly to the active `ThreadImpl`. `consider_yield()` is a no-op under real
OS threading but is load-bearing under `THREAD_SIMPLE_IMPL` — see
Implementation variants below.

The nested `PStatsCallback` class is a hook pair (`activate_hook()` /
`deactivate_hook()`) that PStats installs on a thread so it can attribute CPU
time correctly across context switches, "particularly in the SIMPLE_THREADS
case."

`DEBUG_THREADS` builds add three extra pointer fields
(`_blocked_on_mutex`/`_waiting_on_cvar`/`_waiting_on_cvar_full`) that
[`MutexDebug`](Mutex.md)/[`ConditionVarDebug`](ConditionVar.md)/`ConditionVarFullDebug`
set while this thread is blocked, so `output_blocker()` and the deadlock
detector (see [`Mutex.md`](Mutex.md#implementation-variants)) can report what
a stuck thread is waiting on. The destructor asserts all three are null —
i.e. a `Thread` object must not be destroyed while still recorded as blocked
on something.

## Implementation variants

`Thread` delegates the actual OS-level (or simulated) mechanics to a private
`ThreadImpl _impl` member, whose concrete type is selected at compile time by
`threadImpl.h` based on exactly one of these macros being defined
(`selectThreadImpl.h` picks the macro per platform/build config):

| Macro | Impl class | Behavior |
|---|---|---|
| `THREAD_DUMMY_IMPL` | `ThreadDummyImpl` | Single-threaded build. `start()` always fails; there is no real concurrency. |
| `THREAD_SIMPLE_IMPL` | `ThreadSimpleImpl` | Cooperative user-space threads via setjmp/longjmp context switching (see [`ThreadSimpleManager`](ThreadSimpleManager.md)). Runs on one CPU regardless of thread count. Because switches only happen at explicit yield points, `Mutex`/`ConditionVar` compile down to no-ops in this mode — but **every thread must call `Thread::consider_yield()` occasionally, or it will starve the rest of the running threads** (direct header-comment warning from `threadSimpleImpl.h`). |
| `THREAD_POSIX_IMPL` | `ThreadPosixImpl` | pthreads-backed real OS threads. |
| `THREAD_WIN32_IMPL` | `ThreadWin32Impl` | Native Win32 threads. |

`ThreadDummyImpl`/`ThreadSimpleImpl` further guard their real OS-thread fields
(e.g. `_posix_system_thread_id`, `_win32_system_thread_id` on
`ThreadSimpleImpl`) behind `HAVE_POSIX_THREADS`/`WIN32`, since a
`THREAD_SIMPLE_IMPL` build still runs on top of exactly one real OS thread and
insists that Panda context-switch requests never cross into a different OS
thread than the one it was constructed on — "that's a serious error that may
cause major consequences."

`Thread::is_true_threads()` / `is_simple_threads()` let calling code query
which regime is active at runtime (both false-if `support-threads` is off).

## API reference

```cpp
class Thread : public TypedReferenceCount, public Namable {
protected:
  Thread(const std::string &name, const std::string &sync_name);

PUBLISHED:
  virtual ~Thread();

  static PT(Thread) bind_thread(const std::string &name, const std::string &sync_name);

  const std::string &get_sync_name() const;
  int get_pstats_index() const;
  int get_python_index() const;
  std::string get_unique_id() const;

  int get_pipeline_stage() const;
  void set_pipeline_stage(int pipeline_stage);
  void set_min_pipeline_stage(int min_pipeline_stage);

  static Thread *get_main_thread();
  static Thread *get_external_thread();
  static Thread *get_current_thread();
  static int get_current_pipeline_stage();
  static bool is_threading_supported();
  static bool is_true_threads();
  static bool is_simple_threads();
  BLOCKING static void sleep(double seconds);

  BLOCKING static void force_yield();
  BLOCKING static void consider_yield();

  virtual void output(std::ostream &out) const;
  void output_blocker(std::ostream &out) const;
  static void write_status(std::ostream &out);

  bool is_started() const;
  bool is_joinable() const;

  bool start(ThreadPriority priority, bool joinable);
  BLOCKING void join();
  void preempt();

  TypedReferenceCount *get_current_task() const;
  static void prepare_for_exit();

  class PStatsCallback {
  public:
    virtual ~PStatsCallback();
    virtual void deactivate_hook(Thread *thread);
    virtual void activate_hook(Thread *thread);
  };
};
```

Note: `thread_main()` (the method a subclass overrides) and the constructor
are `protected`; application code never constructs a bare `Thread`.

## Usage

Not constructed directly. Subclass it and override `thread_main()` (as
[`MainThread`](MainThread.md), [`ExternalThread`](ExternalThread.md),
[`GenericThread`](GenericThread.md), and [`PythonThread`](PythonThread.md) all
do), or use `GenericThread` to avoid subclassing entirely:

```cpp
class MyThread : public Thread {
public:
  MyThread() : Thread("my-thread", "my-thread") {}
protected:
  virtual void thread_main() override {
    // ... do work, calling Thread::consider_yield() periodically ...
  }
};

PT(MyThread) t = new MyThread;
t->start(TP_normal, true);
t->join();
```

## Related classes

- [`MainThread`](MainThread.md) — the singleton `Thread` for the process's main thread
- [`ExternalThread`](ExternalThread.md) — the shared `Thread` for unbound non-Panda threads
- [`GenericThread`](GenericThread.md) — spawn a thread from a plain C function pointer
- [`PythonThread`](PythonThread.md) — spawn a thread running a Python callable
- [`ThreadSimpleManager`](ThreadSimpleManager.md) — the scheduler behind `THREAD_SIMPLE_IMPL`
- [`Mutex`](Mutex.md) / [`ConditionVar`](ConditionVar.md) — synchronization primitives whose behavior depends on which `Thread` impl is active
- [`Pipeline`](Pipeline.md) — owner of the pipeline-stage concept a `Thread`'s `_pipeline_stage` indexes into
