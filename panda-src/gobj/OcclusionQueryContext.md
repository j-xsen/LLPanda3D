# OcclusionQueryContext

**Source:** `panda/src/gobj/occlusionQueryContext.h` (+ `.I`, `.cxx`)
**Inherits:** [QueryContext](QueryContext.md) **Inherited by:** (none)

Result handle for a GSG occlusion query — returned in response to a `begin_occlusion_query()`/`end_occlusion_query()` bracket on the GSG, it eventually reports how many fragments (pixels) of the bracketed geometry passed the depth test. Used for occlusion-culling heuristics (e.g. "is this bounding volume actually visible before I draw the real geometry behind it").

## Behavior notes

- Adds exactly one method over `QueryContext`: `get_num_fragments()`, pure virtual (`=0` in the header, base `.cxx` body returning `0` as a fallback, same pattern as `QueryContext::is_answer_ready()`).
- Documented as potentially **blocking** if called before `is_answer_ready()` (inherited from `QueryContext`) returns true — calling it early forces a GPU sync rather than returning garbage.
- Same draw-thread-only restriction as the base class.

## API

| Signature | Notes |
|---|---|
| `virtual int get_num_fragments() const = 0` | Fragments (pixels) that passed the depth test. May block if the answer isn't ready yet. Draw-thread only. |

(Plus everything inherited from [`QueryContext`](QueryContext.md): `is_answer_ready()`, `waiting_for_answer()`.)

## Usage

```cpp
PT(OcclusionQueryContext) query = gsg->end_occlusion_query();
// ... later, same frame or a subsequent one ...
if (query->is_answer_ready()) {
  int visible_pixels = query->get_num_fragments();
  if (visible_pixels == 0) {
    // fully occluded — skip drawing the real geometry
  }
}
```

## See also

- [QueryContext](QueryContext.md) — base class
- [TimerQueryContext](TimerQueryContext.md) — the other concrete query type
