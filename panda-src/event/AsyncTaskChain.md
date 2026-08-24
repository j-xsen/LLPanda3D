# AsyncTaskChain

**Source:** `panda/src/event/asyncTaskChain.h` / `.I` / `.cxx`
**Inherits:** `TypedReferenceCount`, `Namable`

One independently-scheduled queue of tasks, with its own optional pool of
background threads. An [AsyncTaskManager](AsyncTaskManager.md) owns a set of
named chains; tasks run on whichever chain their `get_task_chain()` names
(default `"default"`). Chains are the mechanism for real parallelism — tasks on
*different* chains can run concurrently on separate threads; tasks on the
*same* chain are serialized by sort value (see below) even with multiple
threads assigned to that chain.

## Behavior notes

- **Every task runs exactly once per epoch, by default — priority does not
  change *how often* a task runs, only its order.** With `timeslice_priority`
  false (the default), all tasks run round-robin once per epoch; priority
  only breaks ties in run order within the same `sort` value. Turning
  `timeslice_priority` **on** changes this fundamentally: tasks with priority
  `> 1` get proportionally more runtime (a task with priority 100 gets ~5x
  the runtime of a task with priority 20), and it becomes possible for a
  low-priority task to be skipped entirely in a given epoch.
- **`sort` is a hard barrier; `priority` is a soft ordering hint.** *All*
  tasks at a lower sort value complete (across every thread on the chain)
  before *any* task at a higher sort value begins — this is what
  `finish_sort_group()` enforces internally. Tasks with different priorities
  but the same sort *can* run simultaneously if the chain has multiple
  threads; tasks with different sort values never run simultaneously with
  each other on the same chain.
- **Tie-breaking order** (from `AsyncTaskSortPriority`, used as the active-task
  heap comparator): lowest `sort` value runs first; within equal `sort`,
  highest `priority` first; within equal `priority`, earliest `start_time`
  first; and as a final, always-deterministic tiebreaker, earliest insertion
  order (`_implicit_sort`).
- **0 threads means synchronous/manual polling.** If `num_threads == 0` (the
  default), nothing runs until something calls `poll()` — normally
  `AsyncTaskManager::poll()`, called once per frame by the application
  framework. Setting `num_threads > 0` spawns real background threads that
  service the chain continuously (`AsyncTaskChainThread::thread_main()`);
  `poll()` becomes a no-op in that mode ("This method does nothing in
  threaded mode, so it may safely be called in either case").
- **`frame_budget` only limits `DS_pickup` re-runs, not `DS_cont`.** A
  negative budget (the default, `-1.0`) means unlimited; `>= 0` caps how much
  wall-clock time per frame the chain will spend letting `DS_pickup`-returning
  tasks run again immediately, before deferring the rest to next frame.
- **`frame_sync` only matters for threaded chains.** Non-threaded chains are
  inherently synchronous with the manager's poll cadence already. When true
  on a threaded chain, the chain won't start a new epoch faster than one per
  clock tick even if it finishes early; when false (default), a threaded
  chain immediately loops back to the first task again as soon as it finishes
  the current epoch, regardless of frame boundaries.
- **Changing `num_threads` or `thread_priority` on a running chain stops and
  restarts its threads** (`do_stop_threads()` then, if there are still tasks,
  `do_start_threads()`) — not a live reconfiguration of existing threads.
- **`wait_for_tasks()` blocks the calling thread** until the chain's task list
  is empty — useful for clean shutdown, not for normal per-frame use.
- **An implicit chain is silently auto-created (with a logged warning), never
  a hard error**, whenever a task names a chain that doesn't exist yet — both
  from `AsyncTaskManager::add()` and from a task waking up
  (`AsyncFuture::wake_task()`).

## API

| Signature | Notes |
|---|---|
| `AsyncTaskChain(AsyncTaskManager *manager, const std::string &name)` | Normally created automatically via `AsyncTaskManager::make_task_chain()` |
| `void set_tick_clock(bool)` / `get_tick_clock() const` | Auto-tick the manager's clock once per epoch on this chain |
| `void set_num_threads(int)` / `int get_num_threads() const` | `0` = synchronous/`poll()`-driven |
| `int get_num_running_threads() const` | Actual live thread count; `0` before start or if threading unsupported |
| `void set_thread_priority(ThreadPriority)` / `get_thread_priority() const` | OS thread priority for this chain's threads |
| `void set_frame_budget(double)` / `get_frame_budget() const` | Seconds/epoch for `DS_pickup` re-runs; `< 0` = unlimited |
| `void set_frame_sync(bool)` / `get_frame_sync() const` | Threaded chains only — see "Behavior notes" |
| `void set_timeslice_priority(bool)` / `get_timeslice_priority() const` | Changes what `priority` means — see "Behavior notes" |
| `void stop_threads()` / `void start_threads()` / `bool is_started() const` | Manual thread lifecycle control |
| `bool has_task(AsyncTask*) const` | |
| `void wait_for_tasks()` | Blocks until this chain's task list is empty |
| `int get_num_tasks() const` | |
| `AsyncTaskCollection get_tasks() const` / `get_active_tasks() const` / `get_sleeping_tasks() const` | Point-in-time snapshots |
| `void poll()` | Runs one pass in single-threaded mode; no-op in threaded mode |
| `double get_next_wake_time() const` | Earliest scheduled wake time among sleeping tasks on this chain, or `-1` |
| `void output(std::ostream&) const` / `void write(std::ostream&, int indent_level = 0) const` | |

## Usage

```cpp
AsyncTaskManager *mgr = AsyncTaskManager::get_global_ptr();
AsyncTaskChain *io_chain = mgr->make_task_chain("io");
io_chain->set_num_threads(2);           // background I/O tasks run on 2 threads
io_chain->set_frame_budget(0.005);      // cap DS_pickup re-runs to 5ms/frame

PT(AsyncTask) load_task = new MyLoadTask;
load_task->set_task_chain("io");
mgr->add(load_task);
```

## See also

[AsyncTaskManager.md](AsyncTaskManager.md) (owns a set of these) ·
[AsyncTask.md](AsyncTask.md) (`sort`/`priority`, `DoneStatus`) ·
[AsyncTaskCollection.md](AsyncTaskCollection.md) · [README.md](README.md)
