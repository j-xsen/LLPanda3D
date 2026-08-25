# PipelineCycler&lt;CycleDataType&gt;

**Source:** `panda/src/pipeline/pipelineCycler.{h,I,cxx}`
**Inherits from:** `PipelineCyclerBase` (see Implementation variants)

Maintains up to `n` copies of a [`CycleData`](CycleData.md) page — one per
stage of the [`Pipeline`](Pipeline.md) — so that the head of the pipeline can
freely modify "its" copy while downstream stages (e.g. the render thread)
keep reading an older, unmodified copy, and the copies flow one step
downstream each frame. Declared as a `struct` (not `class`) specifically
"to guarantee byte placement within the object, so that... the inherited
struct's data is likely to be placed by the compiler at the `this` pointer"
— i.e. so a `PipelineCycler<T>*` and a pointer to its base impl are
layout-compatible with minimal indirection.

Access always goes through a **reader or writer helper**, never raw calls on
the cycler itself in application code — see [`CycleDataReader`](CycleDataReader.md),
[`CycleDataLockedReader`](CycleDataLockedReader.md), and
[`CycleDataWriter`](CycleDataWriter.md). "Both kinds of pointers should be
released when you are done, as a sanity check" — the reader/writer classes
handle that automatically via RAII.

## Behavior

`PipelineCycler<T>` itself is a thin template wrapper: under `DO_PIPELINING`
it forwards every call straight to its `PipelineCyclerBase` half; without
`DO_PIPELINING` it instead stores a single `CycleDataType _typed_data` member
directly and each accessor just returns `&_typed_data`, compiling away to
almost nothing.

The `OPEN_ITERATE_*`/`CLOSE_ITERATE_*` macro family (defined in
`pipelineCycler.h`) walks pipeline stages — e.g.
`OPEN_ITERATE_UPSTREAM_ONLY(cycler, current_thread)` loops from
`current_thread->get_pipeline_stage() - 1` down to `0`. They exist "for
updating cache values upstream of the current stage, or for removing bad
pointers from all stages." When `DO_PIPELINING` is undefined the same macros
degenerate to a single iteration with `pipeline_stage = 0` and no lock.

### Implementation variants

`PipelineCyclerBase` (`pipelineCyclerBase.h`) is a typedef selected at
compile time, in priority order:

1. **`PipelineCyclerTrueImpl`** (`THREADED_PIPELINE` defined) — the real
   threaded implementation, described below.
2. **`PipelineCyclerDummyImpl`** (`DO_PIPELINING` defined but `HAVE_THREADS`
   is not) — single-threaded, but adds sanity checks matching the true
   impl's contract (e.g. `read()`/`release_read()` balance, `acquire()`/
   `release()` balance) — "usually the case only in development mode."
3. **`PipelineCyclerTrivialImpl`** (`DO_PIPELINING` undefined) — stores just
   one `CycleData*`, does no locking and no sanity checks at all: "designed
   to do as little as possible, and to compile to nothing, or almost
   nothing."

**`PipelineCyclerTrueImpl`** (`pipelineCyclerTrueImpl.{h,I,cxx}`, inherits
[`PipelineCyclerLinks`](#pipelinecyclerlinks)) holds a `CyclerMutex _lock`
(a `ReMutex` subclass, reentrant so nested read/write acquisitions from the
same thread don't deadlock) and a `CycleDataNode _data[]` array, one slot per
pipeline stage, each tracking a `NPT(CycleData) _cdata` plus a
`_writes_outstanding` count. It registers/unregisters itself with its owning
[`Pipeline`](Pipeline.md) in its constructor/destructor via
`add_cycler()`/`remove_cycler()`.

**Copy-on-write is the core trick**, in `write_stage()`: it only duplicates
the `CycleData` page if this is the *first* write requested for that stage
from the current thread ("we will never have outstanding writes for multiple
threads, because we hold the CyclerMutex during the entire lifetime of
write() .. release()"), **and** only if `old_data->get_node_ref_count() != 1`
— the *node* reference count specifically, not the plain reference count,
because "a standard reference of other than 1 just means that some code
(other than the PipelineCycler) has a pointer, which is safe to modify."
`NodeReferenceCount` is used precisely so the cycler can tell its own
internal references apart from external ones (e.g. a
[`CycleDataReader`](CycleDataReader.md) held elsewhere) when deciding whether
a copy is actually needed. If a copy happens, and more than one stage exists,
the cycler marks itself dirty via `add_dirty_cycler()` so `Pipeline::cycle()`
picks it up next frame. `write_stage_upstream()` is the same idea but walks
backward through earlier stages that share the same pointer, optionally
propagating the new copy all the way back to stage 0 (`force_to_0`) — used,
per the header comment on [`CycleDataStageWriter`](CycleDataWriter.md),
"to recompute a cached value... [that] would go away with the next pipeline
cycle" otherwise.

`cycle()` (called only from `Pipeline::cycle()`, only on dirty cyclers) shifts
every stage's pointer one slot toward the tail — `_data[i] = _data[i-1]` for
`i` from `_num_stages-1` down to `1` — and clears the dirty flag only if every
stage now points at the same object again. The stage that fell off the end is
returned as a `PT(CycleData)` rather than destructed in place, per the
deferred-destruction pattern documented on [`Pipeline`](Pipeline.md#behavior).

`CyclerMutex` exists "solely so we can define the `output()` operator" for
debug printing (`DEBUG_THREADS` only), otherwise behaves exactly like a plain
`ReMutex`.

## PipelineCyclerLinks

`PipelineCyclerLinks` (`pipelineCyclerLinks.{h,I}`) is the base class
supplying `_prev`/`_next` pointers so a `PipelineCyclerTrueImpl` can splice
itself into a `Pipeline`'s clean/dirty linked lists in O(1). It's a
hand-rolled intrusive doubly-linked list rather than an STL container
"because we want PipelineCyclers to be able to add and remove themselves
from this list very quickly." `Pipeline` itself inherits the same pointer
pair (indirectly, as list head) so no separate node type is needed for the
list root. Only compiled when `THREADED_PIPELINE` is defined; otherwise an
empty class.

## API reference

```cpp
template<class CycleDataType>
struct PipelineCycler : public PipelineCyclerBase {
  PipelineCycler(Pipeline *pipeline = nullptr);
  PipelineCycler(CycleDataType &&initial_data, Pipeline *pipeline = nullptr);
  PipelineCycler(const PipelineCycler<CycleDataType> &copy);
  void operator = (const PipelineCycler<CycleDataType> &copy);

  const CycleDataType *read_unlocked(Thread *current_thread) const;
  const CycleDataType *read(Thread *current_thread) const;
  CycleDataType *write(Thread *current_thread);
  CycleDataType *write_upstream(bool force_to_0, Thread *current_thread);
  CycleDataType *elevate_read(const CycleDataType *pointer, Thread *current_thread);
  CycleDataType *elevate_read_upstream(const CycleDataType *pointer, bool force_to_0, Thread *current_thread);

  const CycleDataType *read_stage_unlocked(int pipeline_stage) const;
  const CycleDataType *read_stage(int pipeline_stage, Thread *current_thread) const;
  CycleDataType *write_stage(int pipeline_stage, Thread *current_thread);
  CycleDataType *write_stage_upstream(int pipeline_stage, bool force_to_0, Thread *current_thread);
  CycleDataType *elevate_read_stage(int pipeline_stage, const CycleDataType *pointer, Thread *current_thread);
  CycleDataType *elevate_read_stage_upstream(int pipeline_stage, const CycleDataType *pointer, bool force_to_0, Thread *current_thread);

  CycleDataType *cheat() const;  // bypasses locking/staging entirely, debug use only
};
```

Plus, on `PipelineCyclerTrueImpl` directly: `acquire()`/`release()` (manual
lock control, normally handled by [`CyclerHolder`](CyclerHolder.md)),
`increment_read()`/`release_read()`, `increment_write()`/`release_write()`,
`get_num_stages()`, `get_parent_type()`, `get_read_count()`/`get_write_count()`.

## Usage

Not instantiated directly by application code — a class that needs
per-pipeline-stage double-buffered data declares a nested `CycleData`
subclass and a `PipelineCycler<MyCycleData> _cycler` member, then exposes
access through `CycleDataReader`/`CycleDataWriter` helpers on itself. This
pattern is used pervasively throughout `pgraph`/`gobj` for scene-graph node
and geometry state.

## Related classes

- [`Pipeline`](Pipeline.md) — owns/cycles the collection of all
  `PipelineCyclerTrueImpl` instances
- [`CycleData`](CycleData.md) — the payload type cycled
- [`CycleDataReader`](CycleDataReader.md), [`CycleDataLockedReader`](CycleDataLockedReader.md),
  [`CycleDataWriter`](CycleDataWriter.md) — the RAII accessor classes used to
  actually read/write through a cycler
- [`CyclerHolder`](CyclerHolder.md) — RAII `acquire()`/`release()` wrapper
  used by the `OPEN_ITERATE_*` macros
