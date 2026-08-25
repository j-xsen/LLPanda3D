# Pipeline

**Source:** `panda/src/pipeline/pipeline.{h,I,cxx}`
**Inherits from:** `Namable`

Manages a staged pipeline of data — for instance the render pipeline — so
that each stage of the pipeline can simultaneously access a different copy
of the same data. It owns a collection of [`PipelineCycler`](PipelineCycler.md)
objects (specifically their [`PipelineCyclerTrueImpl`](PipelineCycler.md)
halves) and turns them all at once when `cycle()` is called. There is one
default global instance, the render pipeline (`get_render_pipeline()`);
other specialty pipelines may be constructed as needed.

Everything below `THREADED_PIPELINE` (see [README.md](README.md#core-concepts))
compiles away entirely when it's undefined — the class degenerates to just
tracking `_num_stages` (always 1) and warning if anyone asks for more.

## Behavior

Every attached `PipelineCyclerTrueImpl` lives on one of two intrusive
doubly-linked lists ([`PipelineCyclerLinks`](PipelineCycler.md)) owned by the
`Pipeline`: `_clean` (same value across all pipeline stages) or `_dirty`
(differs across stages, needs cycling). Cyclers move themselves from clean to
dirty by calling `add_dirty_cycler()`; `cycle()` moves cyclers from dirty back
to clean (or leaves them dirty, for `_num_stages > 2`) as it processes them.

`cycle()` "flows all the pipeline data down to the next stage." It swaps
`_dirty` into a local `prev_dirty` list under `_lock`, then — deliberately
**not** holding `_lock` for the rest of the loop, "since it could cause a
deadlock" — walks `prev_dirty` calling each cycler's `cycle()`/`cycle_2()`/
`cycle_3()` (three near-identical hand-specialized loop bodies exist for
`_num_stages` of exactly 2, exactly 3, or the general case, "as an
optimization"). For each cycler it first attempts `cycler->_lock.try_lock()`;
if that fails it skips to the next cycler in the list rather than blocking,
"it's important not to block here in order to prevent one cycler from
deadlocking another" — *unless* it's the last cycler remaining in
`prev_dirty`, in which case it blocks on the real lock anyway, "this is
necessary to trigger the deadlock detection code" (see
[`MutexDebug`](Mutex.md#implementation-variants)). Results of each `cycle*()`
call (a `PT(CycleData)`, the just-superseded copy) are collected into
`saved_cdatas` rather than let destruct immediately, so that any cascading
deletes/side effects they trigger are deferred until after the whole loop —
otherwise a `CycleData` destructor could recursively touch the very lists
`cycle()` is iterating.

`remove_cycler()` has a similar race note: if a cycler is currently dirty and
mid-cycle-owned (`_dirty != 0 && _dirty != _next_cycle_seq`), the caller can't
safely unlink it. It releases both `_lock` and the cycler's own lock, calls
`Thread::force_yield()`, and retries — waiting for `cycle()` to finish with
that cycler before removing it, "so that we can safely remove it."

`set_num_stages()` takes `_cycle_lock` (a `ReMutex`, so it can be re-entered)
plus `_lock`, then locks *every* attached cycler (clean and dirty lists) one
at a time before actually resizing any of them, to guarantee no cycler is
mid-cycle while stage counts change.

`get_render_pipeline()` lazily constructs the singleton via
`make_render_pipeline()`, sizing it from the `pipeline-stages` config
variable (default 1) — "the pipeline can automatically grow stages as needed,
but it will not remove stages automatically."

## API reference

```cpp
Pipeline(const std::string &name, int num_stages);
~Pipeline();

static Pipeline *get_render_pipeline();

void cycle();

void set_num_stages(int num_stages);
void set_min_stages(int min_stages);
int get_num_stages() const;

// Only present when THREADED_PIPELINE is defined:
void add_cycler(PipelineCyclerTrueImpl *cycler);
void add_cycler(PipelineCyclerTrueImpl *cycler, bool dirty);
void add_dirty_cycler(PipelineCyclerTrueImpl *cycler);
void remove_cycler(PipelineCyclerTrueImpl *cycler);
int get_num_cyclers() const;
int get_num_dirty_cyclers() const;

// Only present when THREADED_PIPELINE && DEBUG_THREADS:
typedef void CallbackFunc(TypeHandle type, int count, void *data);
void iterate_all_cycler_types(CallbackFunc *func, void *data) const;
void iterate_dirty_cycler_types(CallbackFunc *func, void *data) const;
```

`add_cycler()`/`add_dirty_cycler()`/`remove_cycler()` are called internally by
`PipelineCyclerTrueImpl`'s constructor/destructor and its dirty-marking
logic — application code does not call these directly.

## Usage

Application code almost never constructs a `Pipeline` directly; use
`Pipeline::get_render_pipeline()` to get the global instance, and
`get_render_pipeline()->cycle()` is called automatically once per frame by
the engine's main loop. A `PipelineCycler<CycleDataType>` registers itself
with a specific `Pipeline` (defaulting to the render pipeline) at
construction.

## Related classes

- [`PipelineCycler`](PipelineCycler.md) — the per-object cycled-data
  container that this class manages in bulk
- [`CycleData`](CycleData.md) — the payload type cycled by each stage
- [`Thread`](Thread.md) — `force_yield()` used during `remove_cycler()`'s
  race-avoidance retry loop
