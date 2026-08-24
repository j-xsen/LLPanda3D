# AsyncTaskCollection

**Source:** `panda/src/event/asyncTaskCollection.h` / `.I` / `.cxx`
**Inherits:** none (standalone value type wrapping a `PTA(PT(AsyncTask))`)
**Inherited by:** [AsyncTaskSequence](AsyncTaskSequence.md) (also uses this as its sub-task list)

A plain, order-preserving list of `AsyncTask` pointers — what
`AsyncTaskChain`/`AsyncTaskManager` query methods (`get_tasks()`,
`get_active_tasks()`, `get_sleeping_tasks()`, `find_tasks()`, ...) return, and
what backs [AsyncTaskSequence](AsyncTaskSequence.md)'s sub-task list.

## Behavior notes

- **Copy-on-write backing array.** Internally a `PTA(PT(AsyncTask))`
  (`PointerToArray`), which is itself refcounted and shareable; `add_task()`,
  `remove_task()`, and `remove_task(index)` all check `_tasks.get_ref_count() > 1`
  first and clone the array before mutating if it's shared, so copying an
  `AsyncTaskCollection` is cheap (shares storage) until one of the copies is
  mutated.
- **Duplicates are allowed by default** — `add_tasks_from()` appends without
  checking for existing entries; `remove_duplicate_tasks()` establishes
  uniqueness explicitly when needed (keeps the *first* occurrence of each
  task, removes later ones).
- **`find_task(name)` is a linear scan**, returning the first match — if
  multiple tasks share a name, the one returned is whatever's first in
  insertion order, not defined by any other criterion.
- **This class itself is not thread-safe** — per the header's own `TODO: None
  of this is thread-safe yet.` comment. `AsyncTaskChain`/`AsyncTaskManager`
  build a *new* `AsyncTaskCollection` under their own lock and hand back a
  private snapshot, so the returned collection is safe to read, but a
  collection actively being built/mutated is not safe to share across
  threads.

## API

| Signature | Notes |
|---|---|
| `void add_task(AsyncTask*)` | |
| `bool remove_task(AsyncTask*)` | By pointer identity; returns false if not found |
| `void remove_task(size_t index)` | By position |
| `void add_tasks_from(const AsyncTaskCollection &other)` | Appends; duplicates allowed |
| `void remove_tasks_from(const AsyncTaskCollection &other)` | Removes every task present in `other` |
| `void remove_duplicate_tasks()` | Keeps first occurrence of each |
| `bool has_task(AsyncTask*) const` | |
| `void clear()` | |
| `AsyncTask *find_task(const std::string &name) const` | First match, or `nullptr` |
| `size_t get_num_tasks() const` / `AsyncTask *get_task(size_t) const` | `MAKE_SEQ`'d as `tasks` in Python |
| `AsyncTask *operator[](size_t) const` | Same as `get_task()` |
| `size_t size() const` | Same as `get_num_tasks()` |
| `void operator+=(const AsyncTaskCollection&)` / `operator+(const AsyncTaskCollection&) const` | Concatenation |
| `void output(std::ostream&) const` / `void write(std::ostream&, int indent_level = 0) const` | One-line count vs. full listing |

## Usage

```cpp
AsyncTaskCollection running = AsyncTaskManager::get_global_ptr()->get_active_tasks();
for (size_t i = 0; i < running.get_num_tasks(); ++i) {
  AsyncTask *task = running.get_task(i);
  // ...
}
```

## See also

[AsyncTask.md](AsyncTask.md) · [AsyncTaskSequence.md](AsyncTaskSequence.md) ·
[AsyncTaskManager.md](AsyncTaskManager.md) (main source of these snapshots) ·
[README.md](README.md)
