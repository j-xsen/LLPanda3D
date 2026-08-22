# AsyncTaskManager

**Source:** `panda/src/event/asyncTaskManager.h` / `.I` / `.cxx`
**Inherits:** `TypedReferenceCount`, `Namable`

The top-level task scheduler: owns a set of named [AsyncTaskChain](AsyncTaskChain.md)s
and is the normal entry point for adding/finding/removing tasks. This is the
C++ engine behind Python's `taskMgr`; `AsyncTaskManager::get_global_ptr()` is
the manager the application framework polls once per frame.

## Behavior notes

- **Always has a `"default"` chain**, created in the constructor
  (`do_make_task_chain("default")`) — a fresh manager is immediately usable
  without any chain setup, single-threaded and synchronous.
- **`add()` calls `upon_birth()` *before* the task is actually queued on its
  chain**, and releases the manager lock while doing so (so a task's
  `upon_birth()` override is free to touch the manager/other tasks without
  deadlocking). Chain lookup/creation and the actual enqueue happen after.
- **Adding a task whose `task_chain` name doesn't exist yet auto-creates it**
  (with a logged warning, not an error) — same behavior as described in
  [AsyncTaskChain.md](AsyncTaskChain.md).
- **`find_task(name)` is O(log n) via a name index** (`_tasks_by_name`, an
  ordered multiset), unlike `AsyncTaskCollection::find_task()`'s linear scan —
  the manager maintains this index incrementally as tasks are
  added/renamed/removed. If multiple tasks share a name, which one
  `find_task()` returns is unspecified/arbitrary among them; use
  `find_tasks(name)` to get all of them.
- **`poll()` iterates every chain by index, not by iterator**, specifically so
  a chain that spawns/removes other chains during its own `do_poll()` doesn't
  invalidate the iteration — and stops early (returns) the moment any chain
  reports `S_interrupted` (i.e. a task on it returned `DS_interrupt`), meaning
  chains *after* the interrupted one in the manager's list are skipped for
  that call to `poll()`, and will simply get their turn on the next call.
- **`remove_task_chain(name)` blocks until that chain's tasks finish** — it
  loops calling `do_wait_for_tasks()` while the chain still has tasks, so
  removing a busy chain is a blocking operation, not instant.
- **`cleanup()` (called from the destructor) forcibly tears down everything**
  — intended for shutdown only, not for routine task removal; it iterates
  chains destructively and expects at most one task to still legitimately
  remain (the one currently executing on the calling thread, if any).
- **The global manager's clock defaults to the global `ClockObject`**, but can
  be swapped via `set_clock()` — this is what governs `set_delay()`/wake-time
  scheduling and (if `set_tick_clock(true)` on a chain) automatic per-epoch
  clock ticking.

## API

### Chains
| Signature | Notes |
|---|---|
| `int get_num_task_chains() const` / `AsyncTaskChain *get_task_chain(int n) const` | `MAKE_SEQ`'d as `task_chains` in Python |
| `AsyncTaskChain *make_task_chain(const std::string &name)` | Returns existing chain if the name is already taken |
| `AsyncTaskChain *find_task_chain(const std::string &name)` | `nullptr` if not found (does not create) |
| `bool remove_task_chain(const std::string &name)` | Blocks until the chain's tasks finish; see "Behavior notes" |

### Tasks
| Signature | Notes |
|---|---|
| `void add(AsyncTask *task)` | Routes to the task's `get_task_chain()`, auto-creating it if needed |
| `bool has_task(AsyncTask*) const` | |
| `AsyncTask *find_task(const std::string &name) const` | O(log n); arbitrary pick among same-named tasks |
| `AsyncTaskCollection find_tasks(const std::string &name) const` | All tasks with that exact name |
| `AsyncTaskCollection find_tasks_matching(const GlobPattern&) const` | Glob-pattern name match |
| `bool remove(AsyncTask*)` | |
| `size_t remove(const AsyncTaskCollection&)` | Bulk remove; returns count actually removed |

### Lifecycle / polling
| Signature | Notes |
|---|---|
| `void set_clock(ClockObject*)` / `ClockObject *get_clock()` | Default: global clock |
| `void wait_for_tasks()` | Blocks until every chain is empty |
| `void stop_threads()` / `void start_threads()` | Applies across all chains |
| `size_t get_num_tasks() const` | Total across all chains |
| `AsyncTaskCollection get_tasks() const` / `get_active_tasks() const` / `get_sleeping_tasks() const` | Point-in-time snapshots, all chains combined |
| `void poll()` | Polls every chain in order; stops early on `DS_interrupt` — see "Behavior notes" |
| `double get_next_wake_time() const` | Earliest wake time across all chains, or `-1` |
| `void cleanup()` | Forcible shutdown teardown; not for routine use |
| `static AsyncTaskManager *get_global_ptr()` | Lazily created singleton |
| `void output(std::ostream&) const` / `void write(std::ostream&, int indent_level = 0) const` | |

## Usage

```cpp
AsyncTaskManager *mgr = AsyncTaskManager::get_global_ptr();

PT(GenericAsyncTask) task = new GenericAsyncTask("update", my_update_func, nullptr);
mgr->add(task);

// Per-frame (normally done by the application framework):
mgr->poll();
```

## See also

[AsyncTaskChain.md](AsyncTaskChain.md) (owned by this class; sort/priority/threading
semantics live there) · [AsyncTask.md](AsyncTask.md) · [AsyncTaskCollection.md](AsyncTaskCollection.md) ·
[README.md](README.md)
