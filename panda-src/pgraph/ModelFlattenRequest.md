# ModelFlattenRequest

**Source:** `panda/src/pgraph/modelFlattenRequest.h` / `.I` / `.cxx`
**Inherits:** [AsyncTask](../event/AsyncTask.md)

An [AsyncTask](../event/AsyncTask.md) that duplicates and flattens
(`NodePath::flatten_strong()`) a subtree in a worker thread, without
disturbing the original — useful for flattening a freshly-loaded model
off the main thread before attaching it to the live scene graph.

## Behavior notes

- Takes its task name from `orig->get_name()` automatically (constructor is
  `explicit ModelFlattenRequest(PandaNode *orig)` — no separate name
  argument).
- `do_task()` wraps `_orig` in a fresh throwaway `NodePath("flatten_root")`
  and calls `flatten_strong()` on it, then stores
  `np.get_child(0).node()` (the flattened result) via `set_result()`.
  **If `_orig` currently has no parents**, it can't simply be
  `attach_new_node()`'d without altering the original (attaching would
  reparent it) — so in that case the request makes a *shallow* copy first
  (`make_copy()` + `copy_children()`) and flattens the copy instead,
  leaving the caller's original untouched either way.

## API

| Method | Notes |
|---|---|
| `explicit ModelFlattenRequest(PandaNode *orig)` | Inline constructor |
| `PandaNode *get_orig() const` | |
| `bool is_ready() const` | `done() && !cancelled()` |
| `PandaNode *get_model() const` | Deprecated — use `result()` |

## Usage

```cpp
PT(ModelFlattenRequest) req = new ModelFlattenRequest(model_node);
AsyncTaskManager::get_global_ptr()->add(req);
// later, once done():
PT(PandaNode) flattened = (PandaNode *)req->result();
```

## See also

- [SceneGraphReducer](SceneGraphReducer.md) (what `flatten_strong()` delegates to), [AsyncTask](../event/AsyncTask.md)
