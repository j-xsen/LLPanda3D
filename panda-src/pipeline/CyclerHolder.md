# CyclerHolder

**Source:** `panda/src/pipeline/cyclerHolder.{h,I,cxx}`
**Inherits from:** none (standalone RAII wrapper)

The [`PipelineCycler`](PipelineCycler.md) equivalent of
[`MutexHolder`](MutexHolder.md): the constructor calls `acquire()` on a
`PipelineCyclerBase`, the destructor calls `release()`.

## Behavior

Compiles to nothing when `DO_PIPELINING` is undefined — with pipelining
disabled, a `PipelineCyclerBase` has no lock to acquire, so `CyclerHolder`
holds no pointer and both its constructor and destructor are empty. This
mirrors how [`MutexHolder`](MutexHolder.md) vanishes when
`HAVE_THREADS`/`DEBUG_THREADS` are both undefined.

## API reference

```cpp
class CyclerHolder {
public:
  CyclerHolder(PipelineCyclerBase &cycler);
  CyclerHolder(const CyclerHolder &copy) = delete;
  ~CyclerHolder();   // calls release()

  CyclerHolder &operator = (const CyclerHolder &copy) = delete;
};
```

## Usage

```cpp
void MyClass::do_something() {
  CyclerHolder holder(_cycler);
  // ... access the cycler's current-stage CycleData safely ...
}
```

In practice, most callers reach for the typed
[`CycleDataReader`](CycleDataReader.md)/[`CycleDataWriter`](CycleDataWriter.md)
family instead, which wrap both the cycler acquire/release *and* the
`begin_read()`/`end_read()` (or `write()`/`release_write()`) calls in one
RAII object. `CyclerHolder` is the lower-level primitive those are built on.

## Related classes

- [`PipelineCycler`](PipelineCycler.md) — the class this holds a lock on
- [`MutexHolder`](MutexHolder.md) — the equivalent wrapper for a plain
  [`Mutex`](Mutex.md)
- [`CycleDataReader`](CycleDataReader.md) / [`CycleDataWriter`](CycleDataWriter.md) — higher-level, typically preferred alternatives
