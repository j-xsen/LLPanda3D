# CullableObject

**Source:** `panda/src/pgraph/cullableObject.h` (+ `.I`, `.cxx`)

The smallest atom of the cull result: one `Geom` plus the `RenderState`/`TransformState` it should be drawn with, optionally with a draw callback instead of a Geom. [CullTraverser](CullTraverser.md) creates one per visible Geom found in a [GeomNode](GeomNode.md) and hands it to [CullHandler::record_object()](CullHandler.md), which takes ownership and must eventually `delete` it. Allocated from a `DELETED_CHAIN` pool (`ALLOC_DELETED_CHAIN`), not plain `new`/`delete`, for allocation-churn performance since one is created per visible Geom per frame.

## Behavior notes

- **`munge_geom()` is where GSG-specific vertex format conversion happens**, run once per object between cull and draw. It: (1) asks the GSG (`get_supported_geom_rendering()`) what point/line rendering features are natively supported, masking off hardware point sprites/perspective points if `hardware-point-sprites`/`hardware-points` config vars disable them; (2) if the geometry uses point-rendering features the GSG can't do natively, calls `munge_points_to_quads()` to convert points into camera-facing quads in software; (3) invokes the `GeomMunger`/`StateMunger` to convert vertex data into the GSG's preferred format; (4) if the vertex data still has CPU-side skinning animation left after munging (i.e. hardware skinning wasn't available), calls `animate_vertices()` to bake it now. If `force` is false and the underlying vertex data isn't resident in memory, returns false instead of blocking — caller should retry later (see `request_resident()`).
- **`draw()` branches on `_draw_callback`:** if set, it sets GSG state/transform, wraps itself in a [GeomDrawCallbackData](GeomDrawCallbackData.md), and invokes the callback instead of drawing the Geom directly — this is the mechanism `CallbackNode`/procedural-geometry nodes use to inject custom OpenGL/DirectX calls into the draw stream. If the callback sets `lost_state`, the GSG is told to forget its cached state afterward (`clear_state_and_transform()`) so the next object doesn't inherit stale assumptions.
- `munge_points_to_quads()` is a substantial software fallback: it converts a `GeomPoints`-style primitive into two triangles per point (camera-facing billboarded quads), handling perspective-scaled point size, per-point rotation, and aspect ratio, and back-to-front sorts the generated quads for correct transparency. This exists purely for GSGs/configs that can't do hardware point sprites.
- `show_vertex_animation` (config var) flashes CPU-animated geometry red and hardware-animated geometry blue by swapping in a static flat-colored, light/texture-off `RenderState` (`get_flash_cpu_state()`/`get_flash_hardware_state()`) every other second — pure debug visualization, has no effect otherwise.
- A `FormatMap` cache (`_format_map`, guarded by a `LightMutex`) memoizes the derived `GeomVertexFormat` used for point-to-quad conversion, keyed by source format + sprite-texcoord + retransform-sprites flags, so the same conversion isn't recomputed every frame for geometry with a stable format.
- `request_resident()` checks both `_geom` and `_munged_data` for residency — used by pager-aware code to decide whether to render this frame or defer.

## API

| Member | Notes |
|---|---|
| `CPT(Geom) _geom` | the geometry to draw (public field, not accessor-wrapped) |
| `CPT(GeomVertexData) _munged_data` | GSG-ready vertex data, filled by `munge_geom()` |
| `CPT(RenderState) _state` | net render state |
| `CPT(TransformState) _internal_transform` | net transform, in the GSG's internal coordinate space |
| `PT(CallbackObject) _draw_callback` | if set, `draw()` invokes this instead of drawing `_geom` |
| `bool munge_geom(gsg, munger, traverser, force)` | see behavior notes; false if not force and data non-resident |
| `void draw(gsg, force, current_thread)` | draws `_geom` or invokes `_draw_callback`, whichever applies |
| `void draw_inline(gsg, force, current_thread)` | assumes GSG state already set; draws the Geom directly, no callback check |
| `void draw_callback(gsg, force, current_thread)` | assumes `_draw_callback` is set; invokes it (crashes if null) |
| `bool request_resident() const` | true if `_geom` and `_munged_data` are both resident |
| `void set_draw_callback(CallbackObject *)` | |
| `static void flush_level()` | flushes the sprite-conversion PStatCollector |
| `void output(std::ostream &) const` | prints `_geom` |

## Usage

Application code doesn't construct `CullableObject`s — [CullTraverser](CullTraverser.md) builds them internally while walking [GeomNode](GeomNode.md)s and passes them to [CullHandler](CullHandler.md)/[CullResult](CullResult.md). Custom `CullHandler` subclasses receive them in `record_object()` and must delete them (or hand ownership to a [CullBin](CullBin.md), as `CullResult` does).

## See also

- [CullHandler](CullHandler.md), [CullResult](CullResult.md), [CullTraverser](CullTraverser.md) — producers/consumers
- [GeomDrawCallbackData](GeomDrawCallbackData.md) — passed to `_draw_callback` when set
- [CullBin](CullBin.md) — where `CullResult` files these once collected
