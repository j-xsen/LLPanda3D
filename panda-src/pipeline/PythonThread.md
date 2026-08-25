# PythonThread

**Source:** `panda/src/pipeline/pythonThread.{h,cxx}`
**Inherits from:** [`Thread`](Thread.md)
**Compiled only when:** `HAVE_PYTHON`

Exposed to Python (`PUBLISHED`) to let Python code spawn a Panda thread that
runs an arbitrary Python callable. `thread_main()` calls
`call_python_func(_function, _args)` and stashes the return value in
`_result`, which `join()` then hands back to the caller.

## Behavior

The constructor `Py_INCREF`s the function object, validates it's callable
(`nassert_raise` if not), and — for pre-3.9 Python built with true OS
threading — calls the now-mostly-vestigial `PyEval_InitThreads()`. The
destructor must re-acquire the GIL before `Py_DECREF`ing its Python object
references, "since the destructor could be called from any thread" — it uses
`PyGILState_Ensure()`/`PyGILState_Release()` around the decrefs (skipped
under `SIMPLE_THREADS`, where GIL state is handled differently — see below).

**`call_python_func()` is genuinely two different implementations** depending
on the active thread implementation, because CPython's GIL machinery assumes
one `PyThreadState` per *OS* thread — an assumption `SIMPLE_THREADS` breaks by
multiplexing several logical Panda threads onto a single OS thread:

- **True OS threads (`else` branch of `#ifdef SIMPLE_THREADS`):** uses the
  standard `PyGILState_Ensure()`/`PyGILState_Release()` pair around the call —
  the normal, textbook approach.
- **`SIMPLE_THREADS`:** cannot use `PyGILState_*` at all ("PyGILState enforces
  policies like only one thread state per OS-level thread, which is not true
  in the case of SIMPLE_THREADS"). Instead it manually swaps `PyThreadState`
  objects with `PyThreadState_Swap()`. The source contains a documented,
  unresolved workaround here: freshly-created `PyThreadState` objects are
  never deleted via `PyThreadState_Delete()` because doing so crashed — "It
  appears that the thread state is still referenced somewhere at the time I
  call delete" — so instead old thread states are pushed onto a static
  `pvector<PyThreadState *>` and recycled on the next call. The comment is
  candid about the fix being empirical: "I wish I understood better what's
  going wrong, but I guess this workaround will do."

In both branches, an exception raised inside the spawned thread's Python code
is deliberately **not** left to propagate silently: `PyErr_Fetch()` grabs it,
it's printed immediately via `PyErr_Print()` (with a `thread_cat.error()` log
line noting which thread it came from) as an on-the-spot callback, and then
restored so the caller (`thread_main()`) sees `result == nullptr` with the
error still set. In the `SIMPLE_THREADS` branch specifically, after handling
the exception it also calls `Thread::get_main_thread()->preempt()` — an
explicit attempt to force the main thread to run next "so it can respond to
the exception immediately," which "only works if the main thread is not
blocked, of course."

Running in the main thread itself (`current_thread == get_main_thread()`)
skips all of the above and just calls the function directly — no GIL
juggling needed since the main thread already holds it.

## API reference

```cpp
class PythonThread : public Thread {
PUBLISHED:
  explicit PythonThread(PyObject *function, PyObject *args,
                        const std::string &name, const std::string &sync_name);
  virtual ~PythonThread();

  BLOCKING PyObject *join();

  MAKE_PROPERTY(args, get_args, set_args);

public:
  PyObject *get_args() const;
  void set_args(PyObject *);
  static PyObject *call_python_func(PyObject *function, PyObject *args);
};
```

`join()` overrides `Thread::join()` to additionally return the Python return
value of the thread function (or `None` if there wasn't one).

## Usage

Not typically constructed from C++ — this class exists specifically for
Python's `threading`-like API (`direct.stdpy.threading` / raw Panda thread
bindings) to spawn a thread running a Python function:

```python
def worker():
    ...

t = PythonThread(worker, None, "worker", "worker")
t.start(TPNormal, True)
result = t.join()
```

## Related classes

- [`Thread`](Thread.md) — base class; see "Implementation variants" for the `SIMPLE_THREADS` vs. true-threads distinction this class works around
- [`GenericThread`](GenericThread.md) — the equivalent for a plain C function pointer instead of a Python callable
