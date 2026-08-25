# LightMutex

**Source:** `panda/src/pipeline/lightMutex.{h,I,cxx}`
**Inherits from:** [`MutexDebug`](Mutex.md#implementation-variants) (if `DEBUG_THREADS`) or [`LightMutexDirect`](#implementation-variants) (otherwise)

A cheaper, non-reentrant mutex, API-identical to [`Mutex`](Mutex.md) but
with one important difference under cooperative `SIMPLE_THREADS`: it
**compiles to a no-op**, performing no locking at all. It's therefore only
safe to protect very small sections of code where the caller is confident no
thread yield can occur mid-section. Under real system threading, `LightMutex`
behaves exactly like `Mutex`.

**`ConditionVar` cannot be used with `LightMutex`** — condition variables
only work with the full [`Mutex`](Mutex.md).

## Behavior

Same shape as `Mutex`: `LightMutex()`'s constructor passes `lightweight =
true` to `MutexDebug` under `DEBUG_THREADS` (vs. `Mutex`'s `false`), which
makes `MutexDebug::do_lock()` take its "not a real mutex, just watch it go
by" branch when `_pstats_count == 0` — i.e. in debug builds, a `LightMutex`
lock is tracked for correctness checking (double-lock detection via
`_missed_threads`) but doesn't actually block anyone, matching the release
build's no-op behavior. See [`Mutex`'s Implementation
variants](Mutex.md#implementation-variants) for `MutexDebug` itself.

## API reference

```cpp
class LightMutex : public MutexDebug /* or LightMutexDirect */ {
PUBLISHED:
  LightMutex();
  explicit LightMutex(const std::string &name);
  LightMutex(const LightMutex &copy) = delete;
  ~LightMutex() = default;

  void operator = (const LightMutex &copy) = delete;

  BLOCKING void acquire() const;
  void release() const;
  bool debug_is_locked() const;
};
```

## Usage

```cpp
LightMutex my_lock;
my_lock.acquire();
// ... very small critical section, no possibility of a thread yield ...
my_lock.release();
```

Prefer [`LightMutexHolder`](MutexHolder.md) for RAII acquire/release.

## Implementation variants

- **`MutexDebug`** — reused from [`Mutex`](Mutex.md#implementation-variants),
  constructed with `lightweight = true` (see Behavior above).
- **`LightMutexDirect`** (`lightMutexDirect.{h,I,cxx}`) — used when
  `!DEBUG_THREADS`. Picks its backing impl type based on `DO_PSTATS`:
  - Normally, `mutable MutexImpl _impl` — the `dtoolbase`-level mutex
    typedef, which is a true no-op (`MutexDummyImpl`) under
    `THREAD_SIMPLE_IMPL`, giving `LightMutex` its documented no-locking
    behavior in cooperative-threading builds.
  - **If `DO_PSTATS` is defined**, `mutable MutexTrueImpl _impl` instead — a
    *real* mutex even under `SIMPLE_THREADS`. The header explains why:
    "any `PStatTimer` call may trigger a context switch, and any low-level
    context switch requires all containing mutexes to be true mutexes." A
    no-op mutex would let a PStats-triggered cooperative switch happen
    mid-"critical section," defeating the purpose of holding a lock at all.

## Related classes

- [`Mutex`](Mutex.md) — full mutex; required if a `ConditionVar` needs to
  wait on it
- [`LightReMutex`](LightReMutex.md) — reentrant counterpart
- [`LightMutexHolder`](MutexHolder.md) — RAII acquire/release wrapper
