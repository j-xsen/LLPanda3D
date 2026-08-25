# ThreadSimpleManager

**Source:** `panda/src/pipeline/threadSimpleManager.{h,I,cxx}`
**Compiled only when:** `THREAD_SIMPLE_IMPL`
**Inherits from:** (standalone singleton)

The global scheduler for all [`ThreadSimpleImpl`](Thread.md#implementation-variants)
objects. This class only exists when Panda is built with the cooperative
"simple threads" implementation (no real OS-level concurrency); it decides
which of the runnable simple threads gets to execute next whenever the
currently-running one yields, sleeps, or blocks. At 845 lines, `.cxx` is the
single largest source file in the `pipeline` module — application code should
never call its methods directly; it's reached through `Thread`'s ordinary
interface (`consider_yield()`, `sleep()`, `join()`, etc.).

## Behavior

**Queues.** The manager tracks threads in several buckets: `_ready` (runnable
this epoch), `_next_ready` (runnable, but deferred to the *next* epoch because
this one already used its timeslice budget), `_sleeping` and `_volunteers`
(both time-ordered min-heaps — the latter for threads that voluntarily
yielded early via `enqueue_ready(thread, /* volunteer */ true)`), `_blocked`
(a map from [`BlockerSimple`](Mutex.md) pointer to the FIFO queue of threads
waiting on it — this is the queue a blocking `Mutex`/`ConditionVar` acquire
enqueues onto), and `_finished` (threads awaiting cleanup — a thread can't
drop its own reference count while running, "since that might deallocate its
own stack," so finished threads are reaped by the manager instead).

**Choosing the next thread (`choose_next_context()`).** When `_ready` is
empty, the manager first tries promoting `_next_ready` into `_ready` (finishing
an "epoch" — one full round where every thread got a chance to run). If both
are empty but there are `_volunteers`, it wakes them (waking volunteers
"politely" yields the whole OS process first via `system_yield()`, so as not
to busy-spin the real CPU while every Panda thread is idle). If nothing at
all is ready and nothing is sleeping or blocked, and this isn't an explicit
shutdown, that's treated as a bug: "No threads are ready to run, but we're
not explicitly shutting down. This is an error condition, an unintentional
deadlock" — it logs `"Deadlock!  All threads blocked.\n"`, calls
`report_deadlock()` to dump each blocked thread's queue, and `abort()`s the
process.

**Timeslice accounting.** Each thread gets a fraction of
`_simple_thread_epoch_timeslice` (default 0.05s) proportional to its priority
weight (`_simple_thread_{low,normal,high,urgent}_weight`) versus how much it
has already run this epoch, tracked via a deque of `TickRecord`s. Several
spots in `do_timeslice_accounting()`/`choose_next_context()` carry an explicit
"Ensure we don't go negative" guard when subtracting a popped tick record's
count from a thread's running total — an edge case the source calls out
directly rather than leaving implicit.

**Yielding the real OS process.** `system_yield()`'s underlying `sleep(0)`
call turned out to be unreliable on modern platforms: "There seem to be some
issues with modern operating systems not wanting to actually yield the
timeslice in response to sleep(0). In particular, Windows and OSX both seemed
to do nothing in that call." The workaround is to force an explicit 1ms sleep
instead (`_simple_thread_yield_sleep`, user-configurable — "though on Windows
that's all the resolution you have"), implemented via `select()` in
`system_sleep()` rather than `sleep()`/`nanosleep()`, since a comment notes
those "don't appear to do the trick" either.

**Circular include.** `threadSimpleManager.h` and `threadSimpleImpl.h` each
`#include` the other at the bottom of the file (marked `/* okcircular */` in
both), since a `ThreadSimpleManager` needs to reference `ThreadSimpleImpl*`
and vice versa. This is a deliberate, acknowledged pattern — not a bug to
"fix" if reorganizing headers in this area.

## Related C primitives: `contextSwitch.h`

`ThreadSimpleImpl`'s actual register-save/restore/stack-switch mechanics live
in `contextSwitch.h` plus one of four platform-specific `.c` source variants
(`contextSwitch_longjmp_src.c`, `contextSwitch_posix_src.c`,
`contextSwitch_ucontext_src.c`, `contextSwitch_windows_src.c`, each
`#include`d by `contextSwitch.c` based on platform). These are deliberately
plain **C, not C++**: "to reduce possible conflicts from longjmp
implementations that attempt to be smart with respect to C++ destructors and
exception handling." The four free functions
(`init_thread_context()`/`save_thread_context()`/`switch_to_thread_context()`/
`alloc_thread_context()`/`free_thread_context()`) are what `ThreadSimpleImpl`
and `ThreadSimpleManager::choose_next_context()` call to actually perform a
context switch between two `ThreadContext` stacks.

## Related class: `BlockerSimple`

`BlockerSimple` (`blockerSimple.{h,I}`, also `THREAD_SIMPLE_IMPL`-only) is the
tiny common base class for `MutexSimpleImpl` and `ConditionVarSimpleImpl` — "a
synchronization primitive that one or more threads might be blocked on." It's
just a `friend class ThreadSimpleManager`-only bitfield (`_flags`, holding a
lock-count for mutexes and a has-waiters bit), used as the key type in
`ThreadSimpleManager::_blocked`. Not meant to be used directly; see
[`Mutex.md`](Mutex.md#implementation-variants) /
[`ConditionVar.md`](ConditionVar.md#implementation-variants) for the classes
built on top of it.

## API reference

```cpp
class ThreadSimpleManager {
public:
  void enqueue_ready(ThreadSimpleImpl *thread, bool volunteer);
  void enqueue_sleep(ThreadSimpleImpl *thread, double seconds);
  void enqueue_block(ThreadSimpleImpl *thread, BlockerSimple *blocker);
  bool unblock_one(BlockerSimple *blocker);
  bool unblock_all(BlockerSimple *blocker);
  void enqueue_finished(ThreadSimpleImpl *thread);
  void preempt(ThreadSimpleImpl *thread);
  void next_context();

  void prepare_for_exit();

  ThreadSimpleImpl *get_current_thread();
  void set_current_thread(ThreadSimpleImpl *current_thread);
  bool is_same_system_thread() const;
  void remove_thread(ThreadSimpleImpl *thread);
  static void system_sleep(double seconds);
  static void system_yield();

  double get_current_time() const;
  static ThreadSimpleManager *get_global_ptr();

  void write_status(std::ostream &out) const;

  // Config variables (public, defined in-class to avoid static-init ordering issues):
  ConfigVariableDouble _simple_thread_epoch_timeslice;
  ConfigVariableDouble _simple_thread_volunteer_delay;
  ConfigVariableDouble _simple_thread_yield_sleep;
  ConfigVariableDouble _simple_thread_window;
  ConfigVariableDouble _simple_thread_low_weight;
  ConfigVariableDouble _simple_thread_normal_weight;
  ConfigVariableDouble _simple_thread_high_weight;
  ConfigVariableDouble _simple_thread_urgent_weight;
};
```

## Usage

Never used directly by application code. Reached internally by
`ThreadSimpleImpl::start()`/`sleep_this()`/`yield_this()` and by
`MutexSimpleImpl`/`ConditionVarSimpleImpl` when they need to block or wake a
thread. Get the singleton via `ThreadSimpleManager::get_global_ptr()` only if
writing low-level thread-implementation code within `pipeline` itself.

## Related classes

- [`Thread`](Thread.md) — see "Implementation variants" for how `ThreadSimpleImpl` fits into the impl-selection scheme this manager serves
- [`Mutex`](Mutex.md) / [`ConditionVar`](ConditionVar.md) — their `*SimpleImpl` variants block/unblock through this manager via `BlockerSimple`
