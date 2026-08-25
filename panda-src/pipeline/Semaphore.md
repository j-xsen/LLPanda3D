# Semaphore

**Source:** `panda/src/pipeline/psemaphore.{h,I,cxx}`
**Inherits from:** none (standalone; owns a [`Mutex`](Mutex.md) + [`ConditionVar`](ConditionVar.md))

A classic counting semaphore. Maintains an internal counter, decremented by
`acquire()` and incremented by `release()`. The counter can never go below
zero — `acquire()` blocks while the count is zero, until another thread
calls `release()`.

Unlike the mutex classes, `Semaphore` is not platform-conditional — it's
implemented once, directly on top of `Mutex` + `ConditionVar`, since a
semaphore is just a guarded counter.

## Behavior

Every operation takes a [`MutexHolder`](MutexHolder.md) on the internal
`_lock` before touching `_count`, so `_count` itself is never accessed
without the lock — including `get_count()`, though the header still warns
"this call is not thread-safe (the count may change at any time)" since the
value can be stale the instant the lock is released.

`acquire()` loops `while (_count <= 0) { _cvar.wait(); }` rather than a plain
`if`, guarding against spurious wakeups (a thread woken by `notify()` must
re-check the condition, since another waiter could have consumed the count
first). `try_acquire()` is the non-blocking version — returns `false`
immediately instead of waiting. `release()` returns the post-increment count,
letting the caller observe the new value without a separate `get_count()`
call (and its associated race).

## API reference

```cpp
class Semaphore {
PUBLISHED:
  explicit Semaphore(int initial_count = 1);
  Semaphore(const Semaphore &copy) = delete;
  ~Semaphore() = default;

  Semaphore &operator = (const Semaphore &copy) = delete;

  BLOCKING void acquire();
  BLOCKING bool try_acquire();
  int release();          // returns the count after incrementing

  int get_count() const;  // not thread-safe, informational only
  void output(std::ostream &out) const;

private:
  Mutex _lock;
  ConditionVar _cvar;
  int _count;
};
```

## Usage

```cpp
Semaphore sem(4);   // allow up to 4 concurrent holders

sem.acquire();
// ... section limited to 4 concurrent threads ...
sem.release();
```

## Related classes

- [`Mutex`](Mutex.md), [`ConditionVar`](ConditionVar.md) — building blocks
- [`MutexHolder`](MutexHolder.md) — used internally for every operation
