# AsyncTaskSequence

**Source:** `panda/src/event/asyncTaskSequence.h` / `.I` / `.cxx`
**Inherits:** [AsyncTask](AsyncTask.md), [AsyncTaskCollection](AsyncTaskCollection.md)

A task that is itself a list of sub-tasks, run one at a time in order, one per
epoch. Conceptually similar to a `Sequence` interval, but arbitrary mid-sequence
jumps are not supported, and (unlike an interval) the sequence's total
duration can change during playback since it's driven by each sub-task's own
`DoneStatus` rather than a fixed timeline.

## Behavior notes

- **Not runnable if empty.** `is_runnable()` returns `get_num_tasks() > 0`
  (inherited list access from `AsyncTaskCollection`) — adding an empty
  sequence to a manager will fail the runnability check.
- **Sub-tasks run with `_state == S_active_nested`**, not `S_active` — they
  are *not* independently visible in the manager's/chain's normal active-task
  bookkeeping; they exist only as the sequence's current sub-task.
- **A sub-task's own `upon_birth()`/`upon_death()` fire as it's entered/left**
  — `set_current_task()` calls them itself (not through the normal
  manager/chain add/remove path), so a sub-task's `done_event`/notify
  callbacks still work correctly even though it's never enqueued on a chain.
- **`DS_again`/`DS_pause` from a sub-task become the *sequence's* delay** — if
  a sub-task returns one of these, the sequence copies that sub-task's
  `_delay`/`_has_delay` onto itself and also returns `DS_again`, effectively
  putting the whole sequence to sleep for that duration. `DS_pause`
  additionally advances `_task_index` so the sequence resumes on the *next*
  sub-task when it wakes (used for the "wait N seconds, then move on" pattern
  — see [AsyncTaskPause](AsyncTask.md#asynctaskpause)).
- **`DS_done` from a sub-task advances `_task_index` and returns
  `DS_cont`** on the sequence, i.e. moves to the next sub-task next epoch.
- **`DS_cont`/`DS_pickup`/`DS_exit`/`DS_interrupt`/`DS_await` pass straight
  through** from the sub-task to the sequence's own return value unmodified.
- **`repeat_count`**: `0` or `1` → run once; `> 1` → run that many full passes
  through the list; negative → loop forever until explicitly removed.
  Decremented once each time the sequence runs off the end of the list (not
  once per sub-task).
- **Running off the end resets `_task_index` to 0 before checking repeat
  count** — so a freshly-emptied sequence (all tasks removed mid-run) that
  still has `_task_index >= get_num_tasks()` correctly reports `DS_done`
  rather than looping on an empty list.

## API

Inherits the full task-list API from [AsyncTaskCollection](AsyncTaskCollection.md)
(`add_task`, `remove_task`, `get_num_tasks`, `get_task`, etc.) plus:

| Signature | Notes |
|---|---|
| `explicit AsyncTaskSequence(const std::string &name)` | |
| `void set_repeat_count(int)` / `int get_repeat_count() const` | See "Behavior notes" |
| `size_t get_current_task_index() const` | Index of the sub-task currently running or about to run |

### Overridden (protected)
`is_runnable()`, `do_task()`, `upon_birth()`, `upon_death()` — see "Behavior notes".

## Usage

```cpp
PT(AsyncTaskSequence) seq = new AsyncTaskSequence("intro");
seq->add_task(new MyFadeInTask);
seq->add_task(new AsyncTaskPause(2.0));   // wait 2 seconds
seq->add_task(new MyShowTextTask);
seq->set_repeat_count(1);
AsyncTaskManager::get_global_ptr()->add(seq);
```

## See also

[AsyncTask.md](AsyncTask.md) (base class; `DoneStatus` semantics) ·
[AsyncTaskCollection.md](AsyncTaskCollection.md) (the list half of this class) ·
[README.md](README.md)
