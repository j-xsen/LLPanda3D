# ExternalThread

**Source:** `panda/src/pipeline/externalThread.{h,cxx}`
**Inherits from:** [`Thread`](Thread.md)

The special "external thread" class, representing "some other, non-Panda
thread." There is one shared global instance, returned by
`Thread::get_external_thread()`, plus one additional instance per thread that
has been explicitly bound via `Thread::bind_thread()`.

## Behavior

Two private constructors (`friend class Thread`): the no-argument form builds
the single shared global instance (named `"External"`/`"External"`, created
lazily by `Thread::init_external_thread()`); the `(name, sync_name)` form is
used by `Thread::bind_thread()` to build a dedicated `ExternalThread` for one
specific external thread that wants its own PStats identity instead of
sharing the global one. Both set `_started = true` immediately, since — like
[`MainThread`](MainThread.md) — the thread this object represents is already
running by the time the object exists. `thread_main()` is a no-op and is
never actually called.

Note from [`Thread.md`](Thread.md): every thread not created through Panda's
own `Thread`/`start()` machinery, and not explicitly bound via
`bind_thread()`, resolves to the *same* shared `ExternalThread` object — so
multiple unrelated external threads will appear as one line in a PStats
graph unless individually bound.

## API reference

```cpp
class ExternalThread : public Thread {
private:
  ExternalThread();
  ExternalThread(const std::string &name, const std::string &sync_name);
  virtual void thread_main();
};
```

No public constructor or additional public API beyond what `Thread` provides.

## Usage

Never constructed directly. The shared instance is obtained via:

```cpp
Thread *ext = Thread::get_external_thread();
```

A per-thread instance is obtained by calling, from within the external thread
itself:

```cpp
PT(Thread) mine = Thread::bind_thread("my-python-thread", "my-python-thread");
```

## Related classes

- [`Thread`](Thread.md) — base class, including `bind_thread()`
- [`MainThread`](MainThread.md) — the analogous singleton for the main thread
