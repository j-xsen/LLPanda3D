# LightReMutex

**Source:** `panda/src/pipeline/lightReMutex.{h,I,cxx}`
**Inherits from:** [`MutexDebug`](Mutex.md#implementation-variants) (if `DEBUG_THREADS`) or [`LightReMutexDirect`](#implementation-variants) (otherwise)

The reentrant counterpart to [`LightMutex`](LightMutex.md) — combines
`LightMutex`'s cheapness (no-op under cooperative `SIMPLE_THREADS`, subject
to the same `DO_PSTATS` exception) with [`ReMutex`](ReMutex.md)'s
same-thread-relock tolerance. Per the header comment, "see `LightMutex` and
`ReMutex`" — it doesn't introduce new concepts of its own.

## Behavior

Same pattern as its siblings: passes `lightweight = true, allow_recursion =
true` to `MutexDebug` under `DEBUG_THREADS`. Under release builds, delegates
to `LightReMutexDirect`.

## API reference

```cpp
class LightReMutex : public MutexDebug /* or LightReMutexDirect */ {
PUBLISHED:
  LightReMutex();
  explicit LightReMutex(const std::string &name);
  LightReMutex(const LightReMutex &copy) = delete;
  ~LightReMutex() = default;

  void operator = (const LightReMutex &copy) = delete;

  BLOCKING void acquire(Thread *current_thread = Thread::get_current_thread()) const;
  void elevate_lock() const;
  void release() const;
  bool debug_is_locked() const;
};
```

## Usage

```cpp
LightReMutex my_lock;
my_lock.acquire();
recursive_helper();   // may re-acquire from the same thread
my_lock.release();
```

Prefer [`LightReMutexHolder`](MutexHolder.md) for RAII acquire/release.

## Implementation variants

- **`MutexDebug`** — reused as in `LightMutex`/`ReMutex`
  (`allow_recursion=true, lightweight=true`).
- **`LightReMutexDirect`** (`lightReMutexDirect.{h,I,cxx}`) — used when
  `!DEBUG_THREADS`. If `HAVE_REMUTEXTRUEIMPL` is available, wraps a
  `mutable ReMutexTrueImpl _impl` directly. Otherwise it doesn't hand-roll
  its own recursive-locking logic the way `ReMutexDirect` does — instead it
  simply wraps `mutable ReMutexDirect _impl` and forwards to it: "If we
  don't have a reentrant mutex, use the one we hand-rolled in
  `ReMutexDirect`" (per the header comment), avoiding duplicating that
  hand-rolled algorithm a second time.

## Related classes

- [`LightMutex`](LightMutex.md) — non-reentrant counterpart
- [`ReMutex`](ReMutex.md) — full (non-light) reentrant mutex
- [`LightReMutexHolder`](MutexHolder.md) — RAII acquire/release wrapper
