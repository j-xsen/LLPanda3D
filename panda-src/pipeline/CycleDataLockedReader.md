# CycleDataLockedReader&lt;CycleDataType&gt;

**Source:** `panda/src/pipeline/cycleDataLockedReader.{h,I,cxx}`, `cycleDataLockedStageReader.{h,I,cxx}`
**Inherits from:** (standalone template, no base)

RAII locked read accessor for a [`CycleData`](CycleData.md) page. The
constructor calls `cycler.read(current_thread)`, which — unlike
[`CycleDataReader`](CycleDataReader.md)'s `read_unlocked()` — acquires and
holds the cycler's lock; the destructor calls `release_read()`. "Since a lock
is held on the data while the instance of this class exists, no other thread
may modify any stage of the pipeline during that time. Thus, this class is
appropriate to use for cases in which you might want to read and then modify
the data."

`CycleDataLockedStageReader<CycleDataType>` is the per-stage variant, taking
an extra `int stage` in its constructor and calling `read_stage()`/
`release_read_stage()`.

## Behavior

The distinguishing feature over `CycleDataReader` is that an instance can be
handed directly to a [`CycleDataWriter`](CycleDataWriter.md) constructor
(`CycleDataWriter(cycler, locked_reader)` or the `take_from` overload), which
"automatically elevates the read lock into a write lock" instead of
releasing and re-acquiring — avoiding a window where another thread could
grab the lock in between. `take_pointer()` exists to support that handoff:
it returns the held pointer and clears the reader's own copy so the
destructor won't also try to release it once ownership has moved to the
writer.

Like all six `CycleData*Reader`/`*Writer` classes, hidden from `interrogate`
via `#ifndef CPPPARSER` (no Python bindings — C++-only, for compile-time and
memory reasons).

## API reference

```cpp
template<class CycleDataType>
class CycleDataLockedReader {
public:
  CycleDataLockedReader(const PipelineCycler<CycleDataType> &cycler,
                        Thread *current_thread = Thread::get_current_thread());
  CycleDataLockedReader(CycleDataLockedReader<CycleDataType> &&from) noexcept;
  ~CycleDataLockedReader();

  const CycleDataType *operator -> () const;
  operator const CycleDataType * () const;

  const CycleDataType *take_pointer();
  Thread *get_current_thread() const;
};

template<class CycleDataType>
class CycleDataLockedStageReader {
public:
  CycleDataLockedStageReader(const PipelineCycler<CycleDataType> &cycler, int stage,
                             Thread *current_thread = Thread::get_current_thread());
  ~CycleDataLockedStageReader();
  const CycleDataType *operator -> () const;
  operator const CycleDataType * () const;
  const CycleDataType *take_pointer();
};
```

## Usage

```cpp
CycleDataLockedReader<CData> cdata(_cycler, current_thread);
if (cdata->needs_update) {
  CycleDataWriter<CData> cdataw(_cycler, cdata);  // elevates the held lock
  cdataw->needs_update = false;
}
```

Held only as long as needed — the lock blocks any other thread's write to
this cycler for the reader's lifetime.

## Related classes

- [`PipelineCycler`](PipelineCycler.md) — supplies `read()`/`release_read()`
- [`CycleDataReader`](CycleDataReader.md) — unlocked variant, faster when no
  subsequent write is planned
- [`CycleDataWriter`](CycleDataWriter.md) — accepts a locked reader to elevate
  into a write lock
