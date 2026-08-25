# MutexHolder / LightMutexHolder / ReMutexHolder / LightReMutexHolder

**Source:** `panda/src/pipeline/{mutexHolder,lightMutexHolder,reMutexHolder,lightReMutexHolder}.{h,I,cxx}`
**Inherits from:** none (standalone RAII wrappers)

Four small, near-identical C++ RAII helpers, one per mutex flavor: the
constructor calls `acquire()`, the destructor calls `release()`. Grouped into
one doc because they're one pattern repeated four times, not four distinct
designs.

| Class | Wraps |
|---|---|
| `MutexHolder` | [`Mutex`](Mutex.md) |
| `LightMutexHolder` | [`LightMutex`](LightMutex.md) |
| `ReMutexHolder` | [`ReMutex`](ReMutex.md) |
| `LightReMutexHolder` | [`LightReMutex`](LightReMutex.md) |

## Behavior

All four compile to nothing (`_mutex` member and every method body vanish)
unless `HAVE_THREADS` or `DEBUG_THREADS` is defined — in a single-threaded
build there's nothing to hold.

`MutexHolder` and `ReMutexHolder` each have a 3rd constructor overload taking
`Mutex *&`/`ReMutex *&` (a reference to a pointer): if the pointer is
`nullptr`, it lazily allocates a new mutex before locking it. Per the source
comment, this exists for "functions that may need to reference a Mutex at
static init time, when it is impossible to guarantee ordering of
initializers" — lets a function-local `static Mutex *my_lock = nullptr;`
guarded by `MutexHolder holder(my_lock);` self-initialize the first time
it's called, sidestepping C++ static-init-order-fiasco risk. `LightMutexHolder`
and `LightReMutexHolder` have the equivalent lazy-pointer overload too.

`ReMutexHolder` and `LightReMutexHolder` additionally accept an optional
`Thread *current_thread` parameter (an optimization to skip a
`Thread::get_current_thread()` call when the caller already has it) — `Mutex`
doesn't actually use this parameter internally ("not actually an
optimization. But we keep this method because it causes a symmetry with
`ReMutexHolder`," per `mutexHolder.I`).

## API reference

```cpp
class MutexHolder {
public:
  MutexHolder(const Mutex &mutex);
  MutexHolder(const Mutex &mutex, Thread *current_thread);
  MutexHolder(Mutex *&mutex);   // lazy-allocates if null
  MutexHolder(const MutexHolder &copy) = delete;
  ~MutexHolder();               // calls release()
  MutexHolder &operator = (const MutexHolder &copy) = delete;
};

class LightMutexHolder {
public:
  LightMutexHolder(const LightMutex &mutex);
  LightMutexHolder(LightMutex *&mutex);
  ~LightMutexHolder();
};

class ReMutexHolder {
public:
  ReMutexHolder(const ReMutex &mutex);
  ReMutexHolder(const ReMutex &mutex, Thread *current_thread);
  ReMutexHolder(ReMutex *&mutex);
  ~ReMutexHolder();
};

class LightReMutexHolder {
public:
  LightReMutexHolder(const LightReMutex &mutex);
  LightReMutexHolder(const LightReMutex &mutex, Thread *current_thread);
  LightReMutexHolder(LightReMutex *&mutex);
  ~LightReMutexHolder();
};
```

All four are non-copyable (copy ctor and `operator=` deleted) — a holder is
tied to exactly one acquire/release pair on the stack.

## Usage

```cpp
void MyClass::do_something() {
  MutexHolder holder(_lock);
  // ... critical section; released automatically on any return path ...
}
```

This is the preferred way to acquire any of the four mutex types in Panda3D
engine code — safer than manual `acquire()`/`release()` because the release
happens even if the function returns early or an exception unwinds through
the scope.

## Related classes

- [`Mutex`](Mutex.md), [`LightMutex`](LightMutex.md), [`ReMutex`](ReMutex.md),
  [`LightReMutex`](LightReMutex.md) — the mutex types these wrap
- [`CyclerHolder`](CyclerHolder.md) — the equivalent RAII wrapper for a
  `PipelineCyclerBase`
