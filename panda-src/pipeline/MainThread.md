# MainThread

**Source:** `panda/src/pipeline/mainThread.{h,cxx}`
**Inherits from:** [`Thread`](Thread.md)

The special "main thread" class. There is exactly one instance in the world,
created lazily and returned by `Thread::get_main_thread()`.

## Behavior

The constructor is `private` (only `Thread` may construct one — `friend class
Thread` — via `Thread::init_main_thread()`, called the first time
`get_main_thread()` runs). It names itself `"Main"` for both the thread name
and sync name, calls `_impl.setup_main_thread()` to tell the active
`ThreadImpl` this object represents the thread that's already running (rather
than one to be spawned), and sets `_started = true` immediately since the
main thread is, definitionally, already started. `thread_main()` is a no-op —
it's never actually invoked, since the main thread doesn't get "started" the
normal way.

## API reference

```cpp
class MainThread : public Thread {
private:
  MainThread();
  virtual void thread_main();
};
```

No public constructor or additional public API beyond what `Thread` provides.

## Usage

Never constructed directly. Obtained via:

```cpp
Thread *main = Thread::get_main_thread();
```

## Related classes

- [`Thread`](Thread.md) — base class
- [`ExternalThread`](ExternalThread.md) — the analogous singleton for non-Panda threads
