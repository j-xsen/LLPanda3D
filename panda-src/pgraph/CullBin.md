# CullBin

**Source:** `panda/src/pgraph/cullBin.h` (+ `.I`, `.cxx`)
**Inherits:** TypedReferenceCount, CullBinEnums
**Inherited by:** `CullBinFixed`, `CullBinStateSorted`, `CullBinBackToFront`, `CullBinFrontToBack`, `CullBinUnsorted` (concrete implementations live in `panda/src/cull`, not this module)

Abstract base for a single named "bin" — a collection of `CullableObject`s (Geom + state) collected for one scene during the cull traversal, later drawn together. [CullTraverser](CullTraverser.md)/[CullResult](CullResult.md) assign each culled object to a bin by name as they walk the scene graph; [CullBinManager](CullBinManager.md) owns the registry of bin names/types/sort order that determines which concrete `CullBin` subclass backs each name.

## Behavior notes

- Pure-virtual `add_object()` and `draw()` mean `CullBin` itself never renders anything — every real bin type (state-sorted, back-to-front, fixed, etc.) is a subclass registered with `CullBinManager` via a `BinConstructor` factory function, so `pgraph` never needs to link against the concrete sorting implementations in `panda/src/cull`.
- `make_next()` defaults to returning `nullptr` (discard everything at end of frame); bin types that want to preserve state across frames (e.g. to detect when nothing changed) override it to return a fresh bin seeded with carried-over data.
- `finish_cull()` is a no-op by default — the hook for a bin type to do post-traversal work like sorting before `draw()`.
- `make_result_graph()` is a debugging/introspection aid, not part of the real render path: it replays every added `CullableObject` into a throwaway `PandaNode`/`GeomNode` tree so tools can inspect what a bin collected. Each *new* `(transform, state)` pair seen during replay starts a new `GeomNode` child — this is purely to make state transitions visible to a human, not a rendering necessity. Never call this on the hot path.
- Flash-color support (`has_flash_color()`/`get_flash_color()`) is only compiled in non-`NDEBUG` builds, driven by a `flash-bin-<name>` config variable read by `CullBinManager::add_bin()`.

## API

| Signature | Notes |
|---|---|
| `CullBin(name, BinType, GraphicsStateGuardianBase *gsg, PStatCollector draw_region_pcollector)` | protected copy ctor also exists for subclass use |
| `const std::string &get_name() const` | |
| `BinType get_bin_type() const` | see [CullBinManager](CullBinManager.md#bintype) |
| `virtual PT(CullBin) make_next() const` | default: returns `nullptr` |
| `virtual void add_object(CullableObject *object, Thread *current_thread) = 0` | called once per culled object during traversal |
| `virtual void finish_cull(SceneSetup *, Thread *)` | default: no-op; post-traversal hook |
| `virtual void draw(bool force, Thread *current_thread) = 0` | issues the actual GSG draw calls |
| `PT(PandaNode) make_result_graph()` | debug/introspection only, see above |
| `bool has_flash_color() const` / `const LColor &get_flash_color() const` | debug builds only |

## Usage

`CullBin` is never constructed directly by application code — it's created by `CullBinManager::make_new_bin()` on behalf of [CullResult](CullResult.md) when a new bin index is first needed. Application code interacts with bins only through `CullBinManager`'s name/type/sort/active setters.

## See also

- [CullBinManager](CullBinManager.md) — owns bin registration, naming, sort order, and the `BinType` enum
- [CullResult](CullResult.md) — the `CullHandler` that creates/looks up `CullBin`s during traversal
- [CullableObject](CullableObject.md) — what gets passed to `add_object()`
