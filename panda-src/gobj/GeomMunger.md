# GeomMunger

**Source:** `panda/src/gobj/geomMunger.h` (+ `.I`, `.cxx`)
**Inherits:** TypedReferenceCount, GeomEnums

Converts vertex data from a [`Geom`](Geom.md) into a format suitable for a
particular rendering backend. Each GSG backend typically subclasses this
(e.g. a `DXGeomMunger`/`GLGeomMunger`) to handle its own vertex-format
quirks (column ordering, packing, required columns). It also applies
runtime render-state effects to vertex data — e.g. scaling color values in
response to a `ColorScaleAttrib` (pgraph, undocumented) — which is why
munging is state-dependent, not purely format-dependent.

## Behavior notes

- **Registration is mandatory and one-way:** a `GeomMunger` is constructed
  unregistered and must be passed through the static `register_munger()`
  before use. Once registered, it is treated as immutable — attempting
  `operator=` on a registered munger asserts. Registration deduplicates:
  all registered mungers that perform the same operation collapse to the
  *same pointer* (via a `pset` keyed by `IndirectCompareTo`, comparing
  through `compare_to_impl()`), so munger identity comparison is
  pointer-equality, same pattern as `RenderState`/`GeomVertexFormat`
  interning. `unregister_mungers_for_gsg()` is called when a GSG shuts down
  to drop every munger registered against it.
- **Two-level result caching:** `munge_format()`/`munge_data()` cache their
  results per `(GeomVertexFormat, GeomVertexAnimationSpec)` in
  `_formats_by_animation`, and `munge_geom()` additionally caches the
  *geom-level* result inside the source `Geom`'s own cache (`Geom::_cache`,
  keyed by `Geom::CacheKey{source_data, munger}` — see
  [Geom.md](Geom.md)'s cache notes), not inside the munger itself. Cache
  entries are `Geom::CacheEntry`/`GeomMunger::CacheEntry`, both
  [`GeomCacheEntry`](GeomCacheEntry.md) subclasses, tracked by the global
  [`GeomCacheManager`](GeomCacheManager.md) LRU.
- **`munge_geom()` freshness check:** a cache hit is only used if
  `geom->get_modified() <= cached_result->get_modified()` **and** the same
  for the vertex data — otherwise it's treated as stale and recomputed
  (there's an acknowledged benign race: two threads recomputing
  simultaneously just both get the same answer).
- **Non-resident data handling:** `munge_geom(geom, data, force, ...)` — if
  `force` is false and either the `Geom` or its vertex data isn't currently
  memory-resident (see [VertexDataPage](VertexDataPage.md) paging), munging
  is skipped and the call returns `false` rather than blocking; `force =
  true` always succeeds but may block on paging the data back in
  (`request_resident()`).
- **Premunge vs. munge:** `premunge_*` is a separate, simpler cache
  (`_premunge_formats`, keyed only by format, no animation/GSG-state
  dependence) used for the `premunge-data` config-var load-time path —
  distinct from the full `munge_*` family which is GSG- and render-state-
  aware and evaluated per-draw.
- Actual conversion logic lives in the protected `*_impl()` virtuals
  (`munge_format_impl()`, `munge_data_impl()`, `munge_geom_impl()`,
  `premunge_format_impl()`, `premunge_data_impl()`, `premunge_geom_impl()`)
  — the base class implementations are identity/no-op; real behavior comes
  from GSG-specific subclasses outside `gobj`.

## API

| Signature | Notes |
|---|---|
| `GeomMunger(GraphicsStateGuardianBase *gsg)` | |
| `GraphicsStateGuardianBase *get_gsg() const` | |
| `bool is_registered() const` | |
| `static PT(GeomMunger) register_munger(GeomMunger *munger, Thread*)` | Returns the canonical (possibly pre-existing) registered instance. |
| `static void unregister_mungers_for_gsg(GraphicsStateGuardianBase*)` | |
| `CPT(GeomVertexFormat) munge_format(const GeomVertexFormat*, const GeomVertexAnimationSpec&) const` | Cached format conversion. |
| `CPT(GeomVertexData) munge_data(const GeomVertexData*) const` | Cached data conversion. |
| `void remove_data(const GeomVertexData*)` | Evict a specific `GeomVertexData`'s cached result. |
| `bool munge_geom(CPT(Geom)&, CPT(GeomVertexData)&, bool force, Thread*)` | Full geom+data munging, in/out params updated in place; see freshness/residency notes above. |
| `CPT(GeomVertexFormat) premunge_format(const GeomVertexFormat*) const` / `premunge_data(...)` / `void premunge_geom(...) const` | Load-time-only variant, no GSG-state dependence. |
| `int compare_to(const GeomMunger&) const` / `geom_compare_to(...) const` | Used for registry dedup / `Geom::CacheKey` ordering. |

## See also

- [Geom](Geom.md) — owns the per-geom munge-result cache this class writes into
- [GeomVertexData](GeomVertexData.md), [GeomVertexFormat](GeomVertexFormat.md),
  [GeomVertexAnimationSpec](GeomVertexAnimationSpec.md)
- [GeomCacheEntry](GeomCacheEntry.md), [GeomCacheManager](GeomCacheManager.md)
