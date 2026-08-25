# Mutex

**Source:** `panda/src/pipeline/pmutex.{h,I,cxx}`
**Inherits from:** [`MutexDebug`](#implementation-variants) (if `DEBUG_THREADS`) or [`MutexDirect`](#implementation-variants) (otherwise)

A standard mutual-exclusion lock. Only one thread can hold ("lock") a `Mutex`
at any given time; other threads trying to acquire it block until the holder
releases it.

`Mutex` is **not reentrant**: a thread must not lock it twice. This may
happen to work on some platforms (e.g. Win32) but will deadlock on others.
Use [`ReMutex`](ReMutex.md) if a thread needs to lock the same mutex more
than once.

## Behavior

`Mutex` itself adds no logic — it's purely a name for "whichever base class
`DEBUG_THREADS` selects" (see Implementation variants below), plus one extra
member: a global static `Mutex::_notify_mutex`, used to serialize Notify
category output (`pipeline_cat`, `thread_cat`, etc.) across threads so log
lines from different threads don't interleave mid-line.

`acquire()`/`release()` are `const` methods, deliberately, "so that you can
lock and unlock const mutexes, mainly to allow thread-safe access to
otherwise const data" (from `mutexDirect.I`). `lock()`/`try_lock()`/
`unlock()` are C++11-style aliases for `acquire()`/`try_acquire()`/
`release()`, letting `Mutex` satisfy `std::lock_guard`/`std::unique_lock`'s
`BasicLockable` concept directly.

## API reference

```cpp
class Mutex : public MutexDebug /* or MutexDirect */ {
PUBLISHED:
  Mutex();
  explicit Mutex(const std::string &name);
  Mutex(const Mutex &copy) = delete;
  ~Mutex() = default;

  void operator = (const Mutex &copy) = delete;

  // Inherited from the selected base (same signature either way):
  BLOCKING void acquire(Thread *current_thread = Thread::get_current_thread()) const;
  BLOCKING bool try_acquire(Thread *current_thread = Thread::get_current_thread()) const;
  void release() const;
  bool debug_is_locked() const;

  void lock();       // C++11-style alias for acquire()
  bool try_lock();
  void unlock();      // alias for release()

public:
  static Mutex _notify_mutex;
};
```

Note: `MutexDirect`'s `acquire()`/`try_acquire()` take no `Thread *`
parameter (release-mode build has no use for it); `MutexDebug`'s do, to
attribute the lock to a thread for deadlock detection. `Mutex`'s own
constructors compile away the name argument entirely under `!DEBUG_THREADS` —
mutex names only exist in debug builds.

## Usage

```cpp
Mutex my_mutex("my_mutex");
my_mutex.acquire();
// ... critical section ...
my_mutex.release();
```

Prefer [`MutexHolder`](MutexHolder.md) (RAII) over manual acquire/release so
the mutex is always released, even if the critical section throws or
returns early.

## Implementation variants

`Mutex`'s base class is chosen entirely at compile time — only one is ever
built into a given binary:

- **`MutexDebug`** (`mutexDebug.{h,I,cxx}`) — used when `DEBUG_THREADS` is
  defined. Implements locking "the hard way" by hand, tracking a
  `_locking_thread`/`_lock_count` pair and a global `MutexTrueImpl
  *_global_lock` that serializes all `MutexDebug` bookkeeping. Adds real
  **deadlock detection**: before blocking, `do_lock()` walks the chain of
  "who is thread X blocked on, and who's *that* mutex's holder blocked on"
  (`next_mutex = next_thread->_blocked_on_mutex`); if the chain loops back to
  the current thread, it calls `report_deadlock()` (dumps the full wait
  chain to `thread_cat.error()`) and raises an assertion instead of hanging.
  Also flags a common use-after-destruct bug: locking a destructed mutex
  hits `nassertd(_lock_count != -100)` (the destructor poisons the count to
  `-100`), and if the `name-deleted-mutexes` config var is set, the
  destructor leaks a copy of the mutex's name specifically so this error
  message can report *which* mutex it was.
- **`MutexDirect`** (`mutexDirect.{h,I,cxx}`) — used when `!DEBUG_THREADS`.
  Thin pass-through: every method forwards straight to a `MutexTrueImpl
  _impl` member, no bookkeeping, no name storage (`set_name()`/`get_name()`
  are no-ops that return an empty string — "the mutex name is only defined
  when compiling in DEBUG_THREADS mode").
- **`MutexTrueImpl`** (`mutexTrueImpl.h`, typedef only) — selects the actual
  low-level OS mutex. Under `THREAD_SIMPLE_IMPL` it's
  [`MutexSimpleImpl`](Thread.md#implementation-variants); otherwise it's
  `MutexImpl` (the platform mutex typedef from `dtoolbase`, e.g. a pthread or
  Win32 critical section). The header explains why this typedef is distinct
  from the similarly-named `MutexImpl` typedef in `dtoolbase`: "we cannot
  define `MutexSimpleImpl` until we have defined the whole
  `ThreadSimpleManager` and related infrastructure" — code living below
  `dtoolbase` (which predates `pipeline`) has to use the dummy/no-op mutex
  under `THREAD_SIMPLE_IMPL`, while code in or above `pipeline` gets the real
  cooperative-yielding one.
- **`MutexSimpleImpl`** (`mutexSimpleImpl.{h,I,cxx}`) — used when
  `THREAD_SIMPLE_IMPL` is defined (cooperative user-space threading, no real
  OS parallelism). "Designed to be as lightweight as possible": `lock()`
  simply yields the calling `ThreadSimpleImpl` to the
  [`ThreadSimpleManager`](ThreadSimpleManager.md) scheduler when the mutex
  would block, rather than doing a real OS wait.

## Related classes

- [`ReMutex`](ReMutex.md) — reentrant variant; use when a thread must lock
  the same mutex twice
- [`LightMutex`](LightMutex.md) — cheaper non-reentrant mutex that
  compiles to a no-op under cooperative `SIMPLE_THREADS`
- [`MutexHolder`](MutexHolder.md) — RAII acquire/release wrapper
- [`ConditionVar`](ConditionVar.md) — waits on a `Mutex`
