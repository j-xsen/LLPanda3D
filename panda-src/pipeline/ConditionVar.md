# ConditionVar

**Source:** `panda/src/pipeline/conditionVar.{h,I,cxx}`
**Inherits from:** `ConditionVarDebug` (if `DEBUG_THREADS`) or [`ConditionVarDirect`](#implementation-variants) (otherwise)

A condition variable used to wake a thread that's waiting for some arbitrary
state change. Constructed against a single [`Mutex`](Mutex.md) (several
condition variables may share the same mutex); `wait()` atomically releases
the mutex and blocks, `notify()` wakes at least one waiter.

`ConditionVar` does **not** implement the full POSIX semantics — critically,
`notify_all()`/broadcast is explicitly deleted (`void notify_all() = delete;`).
Use [`ConditionVarFull`](ConditionVarFull.md) if you need to wake every
waiter at once; `ConditionVar` is kept separate specifically because
`notify_all()` costs more on some platforms (Win32).

Only works with [`Mutex`](Mutex.md), not `LightMutex`/`LightReMutex` — see
[`LightMutex`](LightMutex.md)'s note on why.

## Behavior

`ConditionVar`'s own code is a thin constructor/`get_mutex()` forwarder; the
mutex it's constructed with must outlive it ("It is the caller's
responsibility to ensure the Mutex object does not destruct during the
lifetime of the condition variable" — `conditionVar.I`).

## API reference

```cpp
class ConditionVar : public ConditionVarDebug /* or ConditionVarDirect */ {
PUBLISHED:
  explicit ConditionVar(Mutex &mutex);
  ConditionVar(const ConditionVar &copy) = delete;
  ~ConditionVar() = default;

  ConditionVar &operator = (const ConditionVar &copy) = delete;

  void notify_all() = delete;   // use ConditionVarFull instead

  Mutex &get_mutex() const;

  // Inherited from the selected base:
  BLOCKING void wait();
  BLOCKING void wait(double timeout);
  void notify();
};
```

## Usage

```cpp
Mutex m;
ConditionVar cv(m);

m.acquire();
while (!condition_is_true()) {
  cv.wait();   // atomically releases m, blocks, reacquires m on wakeup
}
// condition holds, m is held
m.release();

// elsewhere, from another thread:
m.acquire();
set_condition_true();
cv.notify();
m.release();
```

## Implementation variants

- **`ConditionVarDebug`** — used under `DEBUG_THREADS`; pairs with
  [`MutexDebug`](Mutex.md#implementation-variants) for consistent
  thread-name/deadlock-aware reporting.
- **`ConditionVarDirect`** (`conditionVarDirect.{h,I,cxx}`) — used when
  `!DEBUG_THREADS`. Thin wrapper storing a reference to the associated
  `MutexDirect &_mutex` plus a `ConditionVarImpl _impl`.
- **`ConditionVarImpl`** (typedef, `conditionVarImpl.h`) — selects the real
  backing primitive per platform/config:
  - **`ConditionVarDummyImpl`** — no-op, used when threading is disabled
    entirely.
  - **`ConditionVarSimpleImpl`** — cooperative-thread version (`:
    BlockerSimple`), used under `THREAD_SIMPLE_IMPL`.
  - **`ConditionVarPosixImpl`** — thin `pthread_cond_t` wrapper.
  - **`ConditionVarWin32Impl`** (`conditionVarWin32Impl.h`) — Windows native
    `Event`-based implementation. Explicitly *simpler* than the `Full`
    variant because it deliberately omits `notify_all()`/broadcast support:
    "the Windows native synchronization primitives don't actually implement
    a full POSIX-style condition variable, but the Event primitive does a
    fair job if we disallow `notify_all()`... This class is much simpler
    than [`ConditionVarFullWin32Impl`], so we can avoid the overhead
    required to support broadcast."
  - **`ConditionVarSpinlockImpl`** (`conditionVarSpinlockImpl.h`) — used
    under `MUTEX_SPINLOCK`. Carries the same warning as
    [`ReMutexSpinlockImpl`](ReMutex.md#implementation-variants): "usually
    not a good idea to use this implementation, unless you are building
    Panda for a specific application on a specific SMP machine, and you are
    confident that you have at least as many CPU's as you have threads."

## Related classes

- [`ConditionVarFull`](ConditionVarFull.md) — adds `notify_all()`, more
  overhead on some platforms
- [`Mutex`](Mutex.md) — the only mutex type a `ConditionVar` can wrap
- [`Semaphore`](Semaphore.md) — built on top of `Mutex` + `ConditionVar`
