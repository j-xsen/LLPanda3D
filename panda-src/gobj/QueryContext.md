# QueryContext

**Source:** `panda/src/gobj/queryContext.h` (+ `.I`, `.cxx`)
**Inherits:** TypedReferenceCount **Inherited by:** [OcclusionQueryContext](OcclusionQueryContext.md), [TimerQueryContext](TimerQueryContext.md)

Base class for asynchronous GPU queries — a round-trip request to the graphics engine whose answer isn't necessarily available immediately. Notably **not** a [`SavedContext`](SavedContext.md) subclass, unlike every other `*Context` class in this module — a deliberate asymmetry called out in the class comment: `SavedContext`s are owned and released by [`PreparedGraphicsObjects`](PreparedGraphicsObjects.md), whereas a `QueryContext` is reference-counted and removes itself from the GSG when the *last reference held by application/caller code* goes away. The caller is responsible for keeping the `QueryContext` pointer alive as long as it still wants the answer.

## Behavior notes

- Pure virtual `is_answer_ready() const` — subclasses must implement the actual poll. The base's own `.cxx` definition (present despite the `=0` in the header — an unusual but valid C++ pattern for pure virtuals with a default body subclasses can still call) returns `false` unconditionally, serving as a fallback/`QueryContext::is_answer_ready()` explicit-base-call target.
- `waiting_for_answer()` is a non-pure virtual hook (default no-op) a caller can invoke to signal to the GSG that it is now blocking on the answer and would like the query expedited — e.g. a GSG backend might perform a driver flush/sync in an override rather than spin-polling.
- Both `is_answer_ready()` and `waiting_for_answer()` are documented as **only valid to call from the draw thread**.

## API

| Signature | Notes |
|---|---|
| `virtual bool is_answer_ready() const = 0` | Poll whether the query result is available yet. Draw-thread only. |
| `virtual void waiting_for_answer()` | Hint to the GSG that the caller is now blocking on the answer. Draw-thread only. |

## Usage

Never constructed directly — see [`OcclusionQueryContext`](OcclusionQueryContext.md) and [`TimerQueryContext`](TimerQueryContext.md) for the concrete query types:

```cpp
PT(OcclusionQueryContext) query = gsg->end_occlusion_query();
// later, on the draw thread:
if (query->is_answer_ready()) {
  int fragments = query->get_num_fragments();
}
```

## See also

- [OcclusionQueryContext](OcclusionQueryContext.md), [TimerQueryContext](TimerQueryContext.md) — concrete subclasses
- [SavedContext](SavedContext.md) — the *other* `*Context` base, contrasted above (ownership model differs)
