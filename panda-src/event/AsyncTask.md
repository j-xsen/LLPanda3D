# AsyncTask

**Source:** `panda/src/event/asyncTask.h` / `.I` / `.cxx` (+ `genericAsyncTask.h/.I/.cxx`, `asyncTaskPause.h/.I/.cxx`)
**Inherits:** [AsyncFuture](AsyncFuture.md), `Namable`
**Inherited by:** `GenericAsyncTask`, `AsyncTaskPause` (both below), [AsyncTaskSequence](AsyncTaskSequence.md)

One schedulable unit of work. To create task behavior, subclass and override
`do_task()` (and optionally `is_runnable()`, `upon_birth()`, `upon_death()`);
or, if you don't want to subclass, use `GenericAsyncTask` with a plain
function pointer. A task is also an [AsyncFuture](AsyncFuture.md) — it can be
awaited, cancelled, and given a `done_event`, on top of running repeatedly.

## Behavior notes

- **`do_task()`'s return value is the entire scheduling contract.** There is
  no separate "reschedule" call — what happens next is determined solely by
  the returned `DoneStatus`:
  - `DS_done` — finished, removed, done callbacks fire.
  - `DS_cont` — keep active, run again next epoch.
  - `DS_again` — like `DS_cont`, but resets `get_start_time()`/`get_elapsed_time()`
    to now/0 (timing *accounting* like `get_dt()` is NOT reset), and re-applies
    `set_delay()` if one is set.
  - `DS_pickup` — like `DS_cont`, but if the chain has spare frame budget
    (`AsyncTaskChain::set_frame_budget()`), run again *this* frame instead of
    waiting.
  - `DS_exit` — stop this task, and (if inside an `AsyncTaskSequence`) stop the
    enclosing sequence too. Outside a sequence, identical to `DS_done`.
  - `DS_pause` — sleep `get_delay()` seconds, then stop. Only meaningful inside
    a sequence (advances to the next sub-task on wake).
  - `DS_interrupt` — abort the *entire* `AsyncTaskManager`'s poll for this
    frame (all chains), but this task itself continues next epoch as if
    `DS_cont`.
  - `DS_await` — suspend until another future completes (see `AsyncFuture`).
- **`cancel()` is `final` and defined as `remove()`.** You cannot override
  cancellation behavior on a task subclass the way you might on a plain
  `AsyncFuture` subclass — removing a task from its manager *is* cancelling it.
- **Sort changes on a currently-queued task can require removal +
  reinsertion.** `set_sort()`/`set_priority()` only trigger a live
  remove/re-add if the task is `S_active` *and* its current sort is still `>=`
  the chain's `_current_sort` (i.e. hasn't already been passed this epoch);
  otherwise the value is just updated in place and takes effect next time it's
  scheduled.
- **`set_task_chain()` on a live, active task also causes an immediate
  remove/re-add** — moving chains isn't just a label change while the task is
  actively queued.
- **`get_name_prefix()` strips trailing `-<digits>`/`_<digits>`** from the
  name (e.g. `"particle-42"` → `"particle"`) to group related tasks under one
  PStats profiling collector. This is a profiling/display concern only —
  actual scheduling order among same-sort tasks is by priority, then start
  time, then insertion order (`AsyncTaskChain::AsyncTaskSortPriority`), never
  by name.
- **`upon_birth()`/`upon_death()` throw generic manager-wide events** —
  `<manager_name>-addTask` / `<manager_name>-removeTask`, each with the task
  itself as the sole `EventParameter`, in addition to whatever your own
  override does (always call the base-class version if you override these).
- **Every task gets a globally unique numeric id** (`get_task_id()`), assigned
  via lock-free compare-and-swap on a static counter at construction time —
  independent of name, which need not be unique.

## GenericAsyncTask

For simple cases that don't warrant a subclass — wraps a plain C function
pointer plus an optional `void*` userdata payload:

```cpp
typedef DoneStatus TaskFunc(GenericAsyncTask *task, void *user_data);
typedef void BirthFunc(GenericAsyncTask *task, void *user_data);
typedef void DeathFunc(GenericAsyncTask *task, bool clean_exit, void *user_data);

GenericAsyncTask(const std::string &name = "");
GenericAsyncTask(const std::string &name, TaskFunc *function, void *user_data);

void set_function(TaskFunc*) / get_function() const;
void set_upon_birth(BirthFunc*) / get_upon_birth() const;   // optional
void set_upon_death(DeathFunc*) / get_upon_death() const;   // optional
void set_user_data(void*) / get_user_data() const;
```

`is_runnable()` returns `_function != nullptr` — a `GenericAsyncTask` with no
function set will refuse to be added to a manager.

```cpp
PT(GenericAsyncTask) task = new GenericAsyncTask("spin");
task->set_function([](GenericAsyncTask *t, void *data) -> AsyncTask::DoneStatus {
  // ... do work ...
  return AsyncTask::DS_cont;
});
AsyncTaskManager::get_global_ptr()->add(task);
```

## AsyncTaskPause

A trivial task whose `do_task()` always returns `DS_pause` — sleeps for the
delay given at construction, then finishes. Exists specifically to insert a
timed pause inside an [AsyncTaskSequence](AsyncTaskSequence.md):

```cpp
AsyncTaskPause(double delay);   // constructor calls set_delay(delay) for you
```

## API

### State
```cpp
enum State { S_inactive, S_active, S_servicing, S_servicing_removed,
             S_sleeping, S_active_nested, S_awaiting };
```
| Signature | Notes |
|---|---|
| `State get_state() const` | |
| `bool is_alive() const` | True for active/servicing/sleeping/active_nested/awaiting; false for inactive/servicing_removed |
| `AsyncTaskManager *get_manager() const` | `nullptr` if `S_inactive` |
| `bool remove()` | Removes from its manager; equivalent to `cancel()` |

### Timing
| Signature | Notes |
|---|---|
| `void set_delay(double)` / `clear_delay()` / `bool has_delay() const` / `double get_delay() const` | Delay before the task first runs (or restarts, on `DS_again`) |
| `double get_wake_time() const` | Only meaningful while `S_sleeping` |
| `void recalc_wake_time()` | Re-applies the current delay immediately, as if the task just returned `DS_again`, while it's sleeping |
| `double get_start_time() const` / `int get_start_frame() const` | Asserts task is not `S_inactive` |
| `double get_elapsed_time() const` / `int get_elapsed_frames() const` | Since start/last `DS_again` |
| `double get_dt() const` / `get_max_dt() const` / `get_average_dt() const` | Wall-clock time consumed by the task's own `do_task()` calls |

### Identity / scheduling
| Signature | Notes |
|---|---|
| `void set_name(const std::string&)` / `clear_name()` | Overridden to keep the manager's name index in sync |
| `std::string get_name_prefix() const` | Name with trailing `-N`/`_N` stripped |
| `AtomicAdjust::Integer get_task_id() const` | Globally unique |
| `void set_task_chain(const std::string&)` / `get_task_chain() const` | Default `"default"`; auto-creates the chain if missing |
| `void set_sort(int)` / `get_sort() const` | Hard ordering group — see [AsyncTaskChain](AsyncTaskChain.md) |
| `void set_priority(int)` / `get_priority() const` | Soft ordering within a sort group |
| `void set_done_event(const std::string&)` | Inherited from `AsyncFuture`; only settable while `S_inactive` |

### Overridable (protected)
| Signature | Notes |
|---|---|
| `virtual bool is_runnable()` | Sanity check before adding to a manager; default `true` |
| `virtual DoneStatus do_task()` | The actual work; default returns `DS_done` |
| `virtual void upon_birth(AsyncTaskManager*)` | Called when added; default throws `<mgr>-addTask` |
| `virtual void upon_death(AsyncTaskManager*, bool clean_exit)` | Called when removed; default throws `<mgr>-removeTask` |

## Usage

```cpp
class SpinTask : public AsyncTask {
public:
  SpinTask() : AsyncTask("spin") {}
protected:
  DoneStatus do_task() override {
    // ... rotate something using get_elapsed_time() ...
    return DS_cont;   // run again next epoch
  }
};

PT(SpinTask) task = new SpinTask;
task->set_delay(1.0);          // wait 1s before first run
AsyncTaskManager::get_global_ptr()->add(task);
```

## See also

[AsyncFuture.md](AsyncFuture.md) (base class) · [AsyncTaskChain.md](AsyncTaskChain.md)
(sort/priority semantics, threading) · [AsyncTaskManager.md](AsyncTaskManager.md) ·
[AsyncTaskSequence.md](AsyncTaskSequence.md) · [README.md](README.md)
