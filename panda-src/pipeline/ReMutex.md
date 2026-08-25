# ReMutex

**Source:** `panda/src/pipeline/reMutex.{h,I,cxx}`
**Inherits from:** [`MutexDebug`](Mutex.md#implementation-variants) (if `DEBUG_THREADS`) or [`ReMutexDirect`](#implementation-variants) (otherwise)

A reentrant mutex: the thread that already holds it may lock it again
without deadlocking. The thread must call `release()` the same number of
times it called `acquire()` before another thread can take the lock.

`ReMutex` reuses `MutexDebug` under `DEBUG_THREADS` (constructed with
`allow_recursion = true`) — the same deadlock-detecting implementation as
[`Mutex`](Mutex.md), just with recursive locking from the same thread
permitted instead of asserting.

## Behavior

Adds nothing over its base beyond the recursion flag — `ReMutex()`'s
constructor passes `true` for `allow_recursion` to `MutexDebug` (vs.
`Mutex`'s `false`). All the interesting logic lives in whichever base is
selected (see Implementation variants).

## API reference

```cpp
class ReMutex : public MutexDebug /* or ReMutexDirect */ {
PUBLISHED:
  ReMutex();
  explicit ReMutex(const std::string &name);
  ReMutex(const ReMutex &copy) = delete;
  ~ReMutex() = default;

  void operator = (const ReMutex &copy) = delete;

  // Inherited:
  BLOCKING void acquire(Thread *current_thread = Thread::get_current_thread()) const;
  BLOCKING bool try_acquire(Thread *current_thread = Thread::get_current_thread()) const;
  void elevate_lock() const;   // increment lock count, assuming already held
  void release() const;
  bool debug_is_locked() const;
};
```

## Usage

```cpp
ReMutex my_lock("my_lock");
my_lock.acquire();
recursive_helper();   // may also acquire() my_lock from the same thread
my_lock.release();
```

Prefer [`ReMutexHolder`](MutexHolder.md) (RAII) for the common case.

## Implementation variants

- **`MutexDebug`** — see [`Mutex`'s Implementation variants](Mutex.md#implementation-variants)
  for the deadlock-detection logic; identical code path, `_allow_recursion`
  just permits the same-thread re-lock branch instead of raising an
  assertion.
- **`ReMutexDirect`** (`reMutexDirect.{h,I,cxx}`) — used when
  `!DEBUG_THREADS`. If the platform provides a true reentrant OS primitive
  (`HAVE_REMUTEXTRUEIMPL` defined), it's a thin pass-through to a
  `ReMutexTrueImpl _impl` member, same shape as `MutexDirect`. Otherwise it
  **hand-rolls reentrancy** on top of a plain `MutexTrueImpl _lock_impl` +
  `ConditionVarImpl _cvar_impl`: `do_lock()` tracks `_locking_thread` /
  `_lock_count` itself, incrementing the count and returning immediately if
  the calling thread already holds it, or blocking on the condvar until the
  holder releases it otherwise — functionally the same algorithm as
  `MutexDebug::do_lock()`'s non-deadlock-checking branch, just without the
  Notify/thread-name overhead. `friend class LightReMutexDirect` — the
  header notes `LightReMutexDirect` reuses this hand-rolled path too.
- **`ReMutexTrueImpl`** — set (via `mutexTrueImpl.h`) to a real platform
  reentrant mutex if `HAVE_REMUTEXIMPL` is available, else to
  **`ReMutexSpinlockImpl`** if `MUTEX_SPINLOCK` is defined, else undefined
  (falls back to the hand-rolled `ReMutexDirect` path above).
- **`ReMutexSpinlockImpl`** (`reMutexSpinlockImpl.{h,I,cxx}`) — a user-space
  spinlock built on `AtomicAdjust`. The header carries an explicit warning:
  "it is usually not a good idea to use this implementation, unless you are
  building Panda for a specific application on a specific SMP machine, and
  you are confident that you have at least as many CPUs as you have
  threads" — a spinning thread burns a full core doing nothing useful if
  the CPU count is lower than the thread count.

## Related classes

- [`Mutex`](Mutex.md) — non-reentrant counterpart
- [`LightReMutex`](LightReMutex.md) — cheaper reentrant mutex, no-op under
  cooperative `SIMPLE_THREADS`
- [`ReMutexHolder`](MutexHolder.md) — RAII acquire/release wrapper
