# TimerQueryContext

**Source:** `panda/src/gobj/timerQueryContext.h` (+ `.I`, `.cxx`)
**Inherits:** [QueryContext](QueryContext.md) **Inherited by:** (none)

Result handle for a GSG GPU-timestamp query — used for precise GPU-side frame timing (e.g. PStats' GPU timing graphs), since CPU-side wall-clock timing can't account for the GPU's own command-queue latency.

## Behavior notes

- Adds `get_timestamp()`, pure virtual, returning a `double`. The doc comment is explicit that there's **no guaranteed clock epoch or units beyond seconds** — only that `end_timestamp - start_timestamp` yields elapsed seconds; don't treat the raw value as meaningful on its own.
- Constructor records `_frame_index` from `ClockObject::get_global_clock()->get_frame_count()` at creation time, plus a caller-supplied `_pstats_index` (both public fields, not accessors) — used to associate a pending query with the specific frame/PStats collector slot that issued it, since the answer may not arrive until several frames later.
- `ALLOC_DELETED_CHAIN(TimerQueryContext)` opts into Panda's pooled-allocator macro for this class — implies these are created/destroyed frequently enough (once or more per frame, per timed GPU region) to be worth a dedicated free-list allocator rather than falling through to the general heap.
- Same draw-thread-only and possible-blocking-before-ready caveats as `OcclusionQueryContext::get_num_fragments()`.

## API

| Signature | Notes |
|---|---|
| `TimerQueryContext(int pstats_index)` | Records the current frame count and the given PStats slot index. |
| `virtual double get_timestamp() const = 0` | GPU timestamp in seconds (relative only — diff two of these). May block if not ready. Draw-thread only. |
| `int _frame_index` | Frame number this query was issued in (public field). |
| `int _pstats_index` | Caller-assigned PStats collector slot (public field). |

(Plus everything inherited from [`QueryContext`](QueryContext.md).)

## Usage

```cpp
PT(TimerQueryContext) start_q = gsg->issue_timer_query(pstats_index);
// ... GPU work happens ...
PT(TimerQueryContext) end_q = gsg->issue_timer_query(pstats_index);
// later, once both are ready:
double gpu_seconds = end_q->get_timestamp() - start_q->get_timestamp();
```

## See also

- [QueryContext](QueryContext.md) — base class
- [OcclusionQueryContext](OcclusionQueryContext.md) — the other concrete query type
