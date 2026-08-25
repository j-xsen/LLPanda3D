# CycleDataReader&lt;CycleDataType&gt;

**Source:** `panda/src/pipeline/cycleDataReader.{h,I,cxx}`, `cycleDataStageReader.{h,I,cxx}`
**Inherits from:** (standalone template, no base)

RAII read-only accessor for a [`CycleData`](CycleData.md) page held by a
[`PipelineCycler`](PipelineCycler.md). The constructor calls
`cycler.read_unlocked(current_thread)` and caches the returned pointer;
`operator->`/`operator const CycleDataType*` expose it. "It is used to access
the data quickly, without holding a lock, for a thread that does not intend
to modify the data and write it back out." If the data might subsequently be
modified, use [`CycleDataLockedReader`](CycleDataLockedReader.md) instead —
`read_unlocked()` gives no protection against another thread writing (and
thus copy-on-write invalidating the pointer) mid-use.

`CycleDataStageReader<CycleDataType>` is the same idea for one specific
pipeline stage: its constructor takes an extra `int stage` and calls
`read_stage_unlocked(stage)` instead of `read_unlocked(current_thread)`.

## Behavior

Both classes are deliberately excluded from `interrogate` bindings —
`#ifndef CPPPARSER` wraps their entire member list — "to improve compile-time
speed and memory utilization." This applies to all six `CycleData*Reader`/
`*Writer` template classes in this module: they have no Python bindings by
design, and are C++-only.

Under `DO_PIPELINING`, `CycleDataReader` stores the cycler pointer, the
current thread, and the resolved data pointer. Without it, `PipelineCycler`
degenerates to holding the `CycleDataType` inline, and the reader's
constructor just calls `cycler.cheat()` to get its address directly — no
cycler/thread bookkeeping needed since there's nothing to cycle.

## API reference

```cpp
template<class CycleDataType>
class CycleDataReader {
public:
  CycleDataReader(const PipelineCycler<CycleDataType> &cycler,
                  Thread *current_thread = Thread::get_current_thread());
  CycleDataReader(const CycleDataReader<CycleDataType> &copy);

  const CycleDataType *operator -> () const;
  operator const CycleDataType * () const;
  const CycleDataType *p() const;

  Thread *get_current_thread() const;
};

template<class CycleDataType>
class CycleDataStageReader {
public:
  CycleDataStageReader(const PipelineCycler<CycleDataType> &cycler, int stage,
                       Thread *current_thread = Thread::get_current_thread());
  const CycleDataType *operator -> () const;
  operator const CycleDataType * () const;
  Thread *get_current_thread() const;
};
```

## Usage

```cpp
CycleDataReader<CData> cdata(_cycler, current_thread);
return cdata->some_field;
```

Constructed on the stack for the duration of a single read; the destructor is
a no-op (no lock was held to release). Never store one longer than the
immediate scope, since the pointer it wraps can be invalidated by a
concurrent write's copy-on-write.

## Related classes

- [`PipelineCycler`](PipelineCycler.md) — supplies `read_unlocked()`/`read_stage_unlocked()`
- [`CycleData`](CycleData.md) — the payload type read
- [`CycleDataLockedReader`](CycleDataLockedReader.md) — locked variant, safe
  when a subsequent write is possible
- [`CycleDataWriter`](CycleDataWriter.md) — read-write counterpart
