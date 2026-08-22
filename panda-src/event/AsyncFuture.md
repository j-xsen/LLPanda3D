# AsyncFuture

**Source:** `panda/src/event/asyncFuture.h` / `.I` / `.cxx`
**Inherits:** `TypedReferenceCount`
**Inherited by:** [AsyncTask](AsyncTask.md), `AsyncGatheringFuture` (below)

A thread-safe promise/future: a handle to a result that isn't ready yet,
which can be waited on, polled, cancelled, or have a callback/event fired when
it resolves. Mirrors Python's `concurrent.futures.Future` / `asyncio.Future`
API closely (per the header, "This API aims to mirror and be compatible with
Python's Future class"). [AsyncTask](AsyncTask.md) *is* an `AsyncFuture` (a
task's completion is itself a future you can await), and
[EventHandler::get_future()](EventHandler.md) returns one that resolves when a
named event fires — this is the bridge between the event system and the task
system.

## Behavior notes

- **State machine has exactly one one-way transition group.** A future starts
  `FS_pending`; `set_result()` moves it to `FS_finished`, `cancel()` moves it
  to `FS_cancelled` — either way `done()` becomes true forever after.
  `FS_locked_pending` is a short-lived internal spinlock state used by
  `try_lock_pending()`/`unlock()` while mutating `_waiting`; application code
  never observes it directly (`done()` treats it as not-done, since it's
  `< FS_finished`).
- **Ownership model: the "resolver thread" owns it until done.** Before
  `done()`, only whichever thread is expected to call `set_result()` should
  touch the result-producing side; other threads may only query state. Once
  done, the result is permanently read-only and safe from any thread.
- **`cancel()` on a task is equivalent to `remove()`** — `AsyncTask` overrides
  `cancel()` as `final` and defines it to call `remove()` (see
  [AsyncTask.md](AsyncTask.md)).
- **`done_event` does not fire on cancellation** — "for historical reasons"
  per the source comment; it fires only on a clean `set_result()`/successful
  finish, not on `cancel()`.
- **`wait()` busy-polls, not condition-variable-blocks.** Both `wait()`
  overloads spin on `Thread::force_yield()` until `done()` (or timeout) — a
  deliberate simplicity-over-efficiency tradeoff per the source comment
  ("It may be more efficient to use a condition variable, but let's not add
  the extra complexity unless we're sure that we need it").
- **`gather(futures)` has three distinct fast paths**: empty list → an
  already-finished future; single-element list → that exact future object
  (not wrapped); 2+ → a real `AsyncGatheringFuture`. Don't assume
  `gather({single_future})` returns a *new* object — it returns
  `single_future` itself.
- **`set_result()` overloads pick the ref-counting story for you** —
  `set_result(TypedObject*)` alone holds no extra reference (the caller must
  keep it alive), while the `TypedReferenceCount*`/`TypedWritableReferenceCount*`/
  `EventParameter` overloads also stash a `PT(ReferenceCount)` so the future
  itself keeps the result alive until it's destroyed.
- **A future must not be destroyed while anything is still waiting on it** —
  the destructor asserts `_waiting.empty()`; it's only valid to destroy a
  future after it's done (and thus has already notified/cleared its waiters).

## AsyncGatheringFuture

A `final` subclass, constructible only via `AsyncFuture::gather()`, that
resolves once every future passed to `gather()` has finished:

```cpp
size_t get_num_futures() const;
AsyncFuture *get_future(size_t i) const;
TypedObject *get_result(size_t i) const;   // result of the i'th input future
```

`cancel()` on the gathering future cancels *every* contained future and waits
for them all to finish before marking itself cancelled, preserving the
guarantee that `result()` is safe once `done()` is true.

## API

| Signature | Notes |
|---|---|
| `bool done() const` | Safe from any thread |
| `bool cancelled() const` | |
| `virtual bool cancel()` | Returns false if already done |
| `void set_done_event(const std::string&)` / `get_done_event() const` | Fires (with `this` as `EventParameter`) on clean finish only, not cancel |
| `void wait()` / `void wait(double timeout)` | Busy-polls; see "Behavior notes" |
| `void set_result(std::nullptr_t)` / `set_result(TypedObject*)` / `set_result(TypedReferenceCount*)` / `set_result(TypedWritableReferenceCount*)` / `set_result(const EventParameter&)` | Ref-counting differs per overload — see "Behavior notes" |
| `TypedObject *get_result() const` | Asserts `done()` |
| `void get_result(TypedObject *&ptr, ReferenceCount *&ref_ptr) const` | Also exposes the held reference, if any |
| `static AsyncFuture *gather(Futures futures)` | See fast-path notes above |
| `virtual bool is_task() const` | `false` here; overridden `true`/`final` in `AsyncTask` |
| `void output(std::ostream&) const` | `TypeName (pending|finished|cancelled)` |

Python-only (not part of the C++ API surface, listed for completeness — see
`asyncFuture_ext.h/.cxx`): `__await__`, `__iter__`, `result(timeout)`,
`add_done_callback(fn)`.

## Usage

```cpp
// A future that resolves on an event:
PT(AsyncFuture) fut = EventHandler::get_global_event_handler()->get_future("level-loaded");
fut->wait();  // or: await it from a coroutine-style task via DS_await

// Gathering several:
AsyncFuture::Futures futs = { task_a, task_b, task_c };
PT(AsyncFuture) all = AsyncFuture::gather(std::move(futs));
all->wait();
```

## See also

[AsyncTask.md](AsyncTask.md) (a future that also does work) ·
[EventHandler.md](EventHandler.md) (`get_future()`) · [README.md](README.md)
