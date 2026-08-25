# CycleDataWriter&lt;CycleDataType&gt;

**Source:** `panda/src/pipeline/cycleDataWriter.{h,I,cxx}`, `cycleDataStageWriter.{h,I,cxx}`
**Inherits from:** (standalone template, no base)

RAII read-write accessor for a [`CycleData`](CycleData.md) page. The
constructor calls `cycler.write(current_thread)` (triggering copy-on-write if
needed, see [`PipelineCycler`](PipelineCycler.md#behavior)); the destructor
calls `release_write()`. `operator->` and `operator CycleDataType*` give
mutable access to the current stage's data for the object's lifetime.

`CycleDataStageWriter<CycleDataType>` is the per-stage variant (extra
`int stage` argument, calls `write_stage()`/`release_write_stage()`).
"Usually this is used to implement writing directly to an upstream pipeline
value, to recompute a cached value there (otherwise, the cached value would
go away with the next pipeline cycle)" — i.e. for lazily-computed cache
fields that must survive a pipeline cycle, write them into an earlier stage
directly rather than letting the normal current-stage write get overwritten
by the next `cycle()`.

## Behavior

Several extra constructor overloads support elevating an existing lock rather
than acquiring a fresh one:

- `CycleDataWriter(cycler, force_to_0, current_thread)` — passes `force_to_0`
  through to `write_upstream()`, propagating the write all the way back to
  stage 0 regardless of existing sharing (see
  [`PipelineCyclerTrueImpl::write_stage_upstream()`](PipelineCycler.md#behavior)).
- `CycleDataWriter(cycler, locked_cdata, current_thread)` / the
  `CycleDataLockedReader&` `take_from` overloads — elevate a held
  [`CycleDataLockedReader`](CycleDataLockedReader.md)'s read lock straight
  into a write lock, "so that we can safely remove it" style race avoidance:
  no window exists where the lock is released and could be grabbed by
  another thread in between the read and the write.

Like the other `CycleData*` accessor templates, hidden from `interrogate`
(`#ifndef CPPPARSER`) — C++-only, no Python bindings, to keep compile time
and memory usage down.

## API reference

```cpp
template<class CycleDataType>
class CycleDataWriter {
public:
  CycleDataWriter(PipelineCycler<CycleDataType> &cycler,
                  Thread *current_thread = Thread::get_current_thread());
  CycleDataWriter(PipelineCycler<CycleDataType> &cycler, bool force_to_0,
                  Thread *current_thread = Thread::get_current_thread());
  CycleDataWriter(PipelineCycler<CycleDataType> &cycler,
                  CycleDataLockedReader<CycleDataType> &take_from);
  CycleDataWriter(PipelineCycler<CycleDataType> &cycler,
                  CycleDataLockedReader<CycleDataType> &take_from, bool force_to_0);
  ~CycleDataWriter();

  CycleDataType *operator -> ();
  const CycleDataType *operator -> () const;
  operator CycleDataType * ();

  Thread *get_current_thread() const;
};

template<class CycleDataType>
class CycleDataStageWriter {
public:
  CycleDataStageWriter(PipelineCycler<CycleDataType> &cycler, int stage,
                       Thread *current_thread = Thread::get_current_thread());
  CycleDataStageWriter(PipelineCycler<CycleDataType> &cycler, int stage,
                       bool force_to_0, Thread *current_thread = Thread::get_current_thread());
  CycleDataStageWriter(PipelineCycler<CycleDataType> &cycler, int stage,
                       CycleDataLockedStageReader<CycleDataType> &take_from);
  ~CycleDataStageWriter();

  CycleDataType *operator -> ();
  operator CycleDataType * ();
};
```

## Usage

```cpp
CycleDataWriter<CData> cdata(_cycler, current_thread);
cdata->some_field = new_value;
```

Held only as long as needed for the modification, since it blocks other
threads from writing the same cycler stage for its lifetime.

## Related classes

- [`PipelineCycler`](PipelineCycler.md) — supplies `write()`/`write_upstream()`/`release_write()`
- [`CycleDataLockedReader`](CycleDataLockedReader.md) — can be elevated into
  a `CycleDataWriter` without releasing the lock in between
- [`CycleData`](CycleData.md) — the payload type written
