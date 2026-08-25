# GenericThread

**Source:** `panda/src/pipeline/genericThread.{h,I,cxx}`
**Inherits from:** [`Thread`](Thread.md)

A generic thread type that lets you spawn a thread from a plain C-style
function pointer plus a `void *` user-data pointer, without having to
subclass `Thread` yourself.

## Behavior

`thread_main()` simply asserts `_function != nullptr` and calls `(*_function)(_user_data)`.
That's the entire implementation — `GenericThread` exists purely to save
callers the boilerplate of writing a one-method `Thread` subclass.

## API reference

```cpp
class GenericThread : public Thread {
public:
  typedef void ThreadFunc(void *user_data);

  GenericThread(const std::string &name, const std::string &sync_name);
  GenericThread(const std::string &name, const std::string &sync_name,
                ThreadFunc *function, void *user_data);

  void set_function(ThreadFunc *function);
  ThreadFunc *get_function() const;

  void set_user_data(void *user_data);
  void *get_user_data() const;
};
```

The two-argument constructor leaves `_function`/`_user_data` null — call
`set_function()`/`set_user_data()` before `start()`ing it.

## Usage

```cpp
void my_thread_func(void *user_data) {
  // ... do work ...
}

PT(GenericThread) t = new GenericThread("worker", "worker",
                                         my_thread_func, nullptr);
t->start(TP_normal, true);
t->join();
```

## Related classes

- [`Thread`](Thread.md) — base class
- [`PythonThread`](PythonThread.md) — the equivalent for a Python callable instead of a C function pointer
