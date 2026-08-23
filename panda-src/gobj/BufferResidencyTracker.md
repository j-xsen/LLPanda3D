# BufferResidencyTracker

**Source:** `panda/src/gobj/bufferResidencyTracker.h` (+ `.I`, `.cxx`)
**Inherits:** (none) **Inherited by:** (none)

Per-`PreparedGraphicsObjects`-family tracker of every [`BufferContext`](BufferContext.md)'s active/resident state, existing to feed Panda's PStats graphics-memory-usage graph. Splits contexts into four [`BufferContextChain`](BufferContextChain.md)s — one per `{active, inactive} × {resident, nonresident}` combination — and reports each chain's byte total to a dedicated `PStatCollector` once per frame.

## Behavior notes

- **The four states, and what they mean for profiling:** `inactive_nonresident` (not drawn, not in video memory — the common/healthy case for unused resources), `active_nonresident` (drawn this frame but the driver reports it's *not* resident — labeled `"Thrashing"` in the PStats collector name, since this typically means the GPU had to fault it in every frame), `inactive_resident` (still occupying video memory despite not being drawn — a GC/eviction candidate), `active_resident` (the steady-state "everything's fine" case).
- **`begin_frame()` only acts once per frame.** It compares the current frame number (`ClockObject::get_global_clock()->get_frame_count()`) against `_active_frame`; if unchanged (already processed this frame), it's a no-op. On a new frame it bulk-demotes both "active" chains into their "inactive" counterparts via `move_inactive()` — every context previously marked active starts the new frame as inactive, and only becomes active again if something calls `set_active(true)` on it (i.e. it gets rendered) during that frame.
- **PStats collector hierarchy:** one shared root collector `"Graphics memory"` (`_gmem_collector`, static — shared across every tracker instance), under which each tracker gets its own named sub-collector (`pgo_name`, e.g. distinguishing GSGs), under which the four states get sub-collectors named by `type_name` (e.g. `"Texture"`, `"Vertex buffer"`) — so PStats can show "Graphics memory > GSG-0 > Active > Texture" style breakdowns. Collector name order is deliberately reversed from state-check order because "these are ordered in reverse order that we would like them to appear in the PStats graph" (per the header comment).
- `end_frame()`/`set_levels()` are functionally identical (`set_levels()` exists to let something reset the reported levels mid-frame without waiting for the natural end-of-frame call).

## API

| Signature | Notes |
|---|---|
| `BufferResidencyTracker(const std::string &pgo_name, const std::string &type_name)` | Sets up the PStatCollector hierarchy for one GSG × one buffer-type combination. |
| `void begin_frame(Thread *current_thread)` | Once per frame: demotes previous "active" chains to "inactive". |
| `void end_frame(Thread *current_thread)` | Pushes each chain's current byte total to its PStatCollector. |
| `void set_levels()` | Same PStats push as `end_frame()`, usable mid-frame. |
| `BufferContextChain &get_inactive_nonresident()` / `get_active_nonresident()` / `get_inactive_resident()` / `get_active_resident()` | Direct access to the four chains (used by `BufferContext::set_owning_chain()` via the `_chains[]` array). |
| `void write(std::ostream &out, int indent_level) const` | Debug dump of every non-empty chain. |

## Usage

One tracker is typically owned per resource-type-per-GSG (e.g. inside `PreparedGraphicsObjects`), driven once per frame:

```cpp
residency_tracker.begin_frame(current_thread);
// ... rendering happens; various BufferContext::set_active(true) calls occur ...
residency_tracker.end_frame(current_thread);
```

## See also

- [BufferContext](BufferContext.md) — the objects being tracked; owns the `_residency_state` bits this class interprets
- [BufferContextChain](BufferContextChain.md) — the four per-state lists this class manages
- [PreparedGraphicsObjects](PreparedGraphicsObjects.md) — typical owner of a `BufferResidencyTracker` per resource type
