# ConditionVarFull

**Source:** `panda/src/pipeline/conditionVarFull.{h,I,cxx}`
**Inherits from:** `ConditionVarFullDebug` (if `DEBUG_THREADS`) or [`ConditionVarFullDirect`](#implementation-variants) (otherwise)

The full-semantics counterpart to [`ConditionVar`](ConditionVar.md): adds
`notify_all()`, guaranteed to wake every thread currently waiting (`notify()`
only guarantees at least one). Exists as a separate class specifically
because "on certain platforms (e.g. Win32), implementing `notify_all()`
requires more overhead" (per the header) — code that never needs to wake
everyone at once should use the cheaper `ConditionVar` instead.

Even `ConditionVarFull` doesn't implement full POSIX semantics: unlike
`pthread_cond_broadcast`, this implementation *requires* (not just permits)
the caller to be holding the associated mutex when calling `notify()` or
`notify_all()`.

## Behavior

Like `ConditionVar`, `ConditionVarFull` itself is just a constructor/
`get_mutex()` forwarder — the substantive logic lives in the selected
Debug/Direct base and, below that, the platform impl.

## API reference

```cpp
class ConditionVarFull : public ConditionVarFullDebug /* or ConditionVarFullDirect */ {
PUBLISHED:
  explicit ConditionVarFull(Mutex &mutex);
  ConditionVarFull(const ConditionVarFull &copy) = delete;
  ~ConditionVarFull() = default;

  ConditionVarFull &operator = (const ConditionVarFull &copy) = delete;

  Mutex &get_mutex() const;

  // Inherited:
  BLOCKING void wait();
  BLOCKING void wait(double timeout);
  void notify();
  void notify_all();   // wakes every waiter; caller must hold the mutex
};
```

## Usage

```cpp
Mutex m;
ConditionVarFull cv(m);

m.acquire();
set_shared_state();
cv.notify_all();   // must hold m
m.release();
```

## Implementation variants

- **`ConditionVarFullDebug`** — debug-mode pairing with
  [`MutexDebug`](Mutex.md#implementation-variants).
- **`ConditionVarFullDirect`** (`conditionVarFullDirect.{h,I,cxx}`) — release
  build; wraps a `ConditionVarFullImpl _impl` plus a `MutexDirect &_mutex`
  reference.
- **`ConditionVarFullImpl`** (typedef) — per-platform backing:
  - **`ConditionVarFullWin32Impl`** (`conditionVarFullWin32Impl.h`) — used
    under `_WIN32`. Implements the ["SetEvent" pattern described by a
    Douglas Schmidt
    article](http://www.cs.wustl.edu/~schmidt/win32-cv-1.html) to support
    both `notify()` and `notify_all()` on top of Win32 `Event`s — more
    overhead than the plain `ConditionVarWin32Impl` (see
    [`ConditionVar`'s Implementation
    variants](ConditionVar.md#implementation-variants)). The header
    explicitly flags known weaknesses inherited from that pattern: "it does
    not necessarily wake up all threads fairly; and it may sometimes
    incorrectly wake up a thread that was not waiting at the time
    `notify()` was called. But we figure it's good enough for our
    purposes."
  - On other platforms, the `Full` variant typically maps to the same
    underlying primitive family as plain `ConditionVar` (Posix/Simple/
    Spinlock/Dummy) — those primitives natively support broadcast, so no
    separate Full-specific class is needed outside the Win32 case.

## Related classes

- [`ConditionVar`](ConditionVar.md) — cheaper, `notify()`-only variant;
  prefer this when `notify_all()` isn't needed
- [`Mutex`](Mutex.md) — the mutex type a `ConditionVarFull` wraps
