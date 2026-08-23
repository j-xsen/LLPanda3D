# CullResult

**Source:** `panda/src/pgraph/cullResult.h` (+ `.I`, `.cxx`)
**Inherits:** ReferenceCount

The standard, only-substantial [CullHandler](CullHandler.md) implementation: sorts every [CullableObject](CullableObject.md) it receives into the appropriate [CullBin](CullBin.md) (by `RenderState::get_bin_index()`), and drives per-frame munging, wireframe/transparency-mode expansion, and debug flash-color overrides along the way. One `CullResult` is created per `DisplayRegion` per frame by the display layer; `make_next()` lets bin state persist across frames.

## Behavior notes

- **`add_object()` is the busiest method in the module** — before an object reaches its bin it may be rewritten or even split into multiple objects:
  - **Auto normal rescale:** if `RescaleNormalAttrib::M_auto` is in effect, picks `M_none`/`M_rescale`/`M_normalize` based on whether `_internal_transform` has identity, uniform, or non-uniform scale (rescale is cheap, full renormalize is needed only for non-uniform scale).
  - **`RenderModeAttrib::M_filled_wireframe`:** clones the object, gives the clone a wireframe overlay state (composed with the object's `ShaderAttrib` if any, via `get_wireframe_overlay_state()`), munges and files the clone into the `"fixed"` bin, then continues rendering the original filled — i.e. one input object becomes two draw calls (filled + wireframe overlay).
  - **Transparency mode dispatch** on `TransparencyAttrib`: `M_alpha`/`M_premultiplied_alpha` compose in an alpha-test state (skip zero-alpha pixels); `M_binary` composes an explicit alpha-test threshold state; `M_multisample`/`M_multisample_mask` fall back to the `M_binary` state if `gsg->get_supports_multisample()` is false; **`M_dual`** is the most involved — it's implemented by literally duplicating the object: a transparent-blended copy goes into the transparent bin (respecting `m_dual_transparent`) and the original, recomposed with an opaque-alpha-test state, continues into its normal (opaque) bin (respecting `m_dual_opaque`) — but **only if no explicit bin was already set on the object** (a `CullBinAttrib` present with a non-empty bin name disables the dual-copy and M_dual degrades to plain M_alpha). All of the debug flash-color checks (`show-transparency`, `flash-bin-<name>`) are non-`NDEBUG`-only and layer a config-driven flat color onto the state for visual debugging.
  - Finally looks up the object's bin by `get_bin_index()`, calls `munge_geom()` (see [CullableObject](CullableObject.md)) — **if munging fails (data not resident and not forced), the object is silently dropped this frame** (deleted, not added to any bin) rather than retried — and hands it to `CullBin::add_object()`.
- **`make_next()`** is the frame-to-frame bin persistence mechanism: it creates a fresh `CullResult` and, for each existing bin, calls `old_bin->make_next()` to carry forward whatever the bin type wants to keep — *unless* the bin's type was changed via `CullBinManager` since last frame (`old_bin->get_bin_type() != bin_manager->get_bin_type(i)`), in which case the slot is reset to `nullptr` and a fresh bin will be created on demand.
- **`finish_cull()`** additionally purges any bin whose `CullBinManager::get_bin_active()` is now false — an inactive bin isn't sorted, drawn, *or carried forward*; its slot is cleared outright.
- **`draw()`** iterates bins strictly in `CullBinManager`'s sorted order (not creation order or bin-index order), wrapping each bin's draw calls in a GSG "group marker" (`push_group_marker()`/`pop_group_marker()`, consumed by graphics-debugger tools like RenderDoc/PIX) named after the bin.
- `bin_removed()` is a stub — `nassertv(false)`; despite the doc comment claiming it should update per-`CullResult` bin caches when a bin is removed via `CullBinManager::remove_bin()`, it's unimplemented in this version. Calling `CullBinManager::remove_bin()` while any `CullResult` holds a cached bin for that index will hit this assert in debug builds.
- `make_result_graph()` mirrors [CullBin::make_result_graph()](CullBin.md) one level up — builds a debug scene graph with one child node per active bin (in sorted draw order), each containing that bin's `make_result_graph()` output. Debug/introspection only.

## API

| Signature | Notes |
|---|---|
| `CullResult(GraphicsStateGuardianBase *gsg, const PStatCollector &draw_region_pcollector)` | |
| `PT(CullResult) make_next() const` | carries forward reusable bin state for next frame |
| `CullBin *get_bin(int bin_index)` | creates the bin on first request (via `CullBinManager::make_new_bin()`) |
| `void add_object(CullableObject *object, const CullTraverser *traverser)` | takes ownership; see behavior notes |
| `void finish_cull(SceneSetup *scene_setup, Thread *current_thread)` | call once traversal is done, before `draw()` |
| `void draw(Thread *current_thread)` | draws all active bins in `CullBinManager` sort order |
| `PT(PandaNode) make_result_graph()` | debug/introspection only |
| `static void bin_removed(int bin_index)` | unimplemented stub, see behavior notes |

## Usage

```cpp
CullResult *result = new CullResult(gsg, draw_region_pcollector);
CullTraverser trav;
trav.set_cull_handler(result);
// ... set_scene(), set_view_frustum(), traverse() ...
trav.end_traverse();
result->finish_cull(scene_setup, current_thread);
result->draw(current_thread);
```

## See also

- [CullHandler](CullHandler.md) — the interface this implements
- [CullBin](CullBin.md), [CullBinManager](CullBinManager.md) — where objects end up and draw-order is decided
- [CullTraverser](CullTraverser.md), [CullableObject](CullableObject.md) — producer and payload
