# RenderState

**Source:** `panda/src/pgraph/renderState.h` (+ `.I`, `.cxx`)
**Inherits:** NodeCachedReferenceCount

An immutable, automatically-interned set of at most one [RenderAttrib](RenderAttrib.md)
per attribute slot, representing a node's complete local rendering state
(color, texture, transparency, depth test, …). Every [PandaNode](PandaNode.md)
carries a `CPT(RenderState)`; the cull traverser composes states down the
graph to determine what each Geom actually renders with. Never constructed
or mutated directly — always obtained via `make()`/`add_attrib()`/etc.,
which return a shared, cached pointer.

## Behavior notes

- **Interning ("uniquification"):** every factory method funnels through
  `return_new()`, which either returns the object as-is (if `uniquify_states`
  is off and the state is non-empty) or looks it up in a global content-hash
  table via `return_unique()` and returns the existing shared instance
  instead, so two `RenderState`s built from the same attribs+overrides
  become the same pointer. This is what makes state comparison an O(1)
  pointer-equality check almost everywhere in the renderer.
- **Empty state:** `make_empty()`/`_empty_state` is the canonical identity
  state (`is_empty()` checks `_filled_slots.is_zero()`); `compose()` and
  `invert_compose()` special-case it as a fast identity no-op.
- **compose()/invert_compose() are cached per pair:** the result of
  composing `this` with `other` is memoized in `_composition_cache` (and
  the mirror entry in `other`'s own cache, so either object's destruction
  invalidates the pair). Disable via the `state-cache` config variable.
- **Per-slot composition rule (`do_compose`)**, for each attrib slot where
  either side has an attrib (`a` = this, `b` = other):
  - if only one side has an attrib, that side wins outright (no `compose()`
    call, so a "later" empty slot never suppresses an earlier attrib);
  - if `b`'s override is lower than `a`'s, `a` wins outright;
  - if `a`'s override is lower than `b`'s, normally `a` still wins **unless**
    `a`'s attrib overrides `RenderAttrib::lower_attrib_can_override()` to
    return true (a handful of attribs opt into "even a lower-priority
    instance of me should win" semantics);
  - otherwise (equal override, or the override rule above didn't apply),
    the two attribs are combined via `a->compose(b)`, taking `b`'s override
    value.
- **`add_attrib(attrib, override)` respects override priority**: if the slot
  already has an attrib with a *strictly higher* override value, the call is
  a no-op and returns `this` unchanged. `set_attrib()` (with or without an
  explicit override) always replaces unconditionally, ignoring priority —
  used to force a value regardless of what is already there.
- **Garbage collection:** interned states are refcounted three ways
  (`cache_ref`/`node_ref`/plain `ref`); `RenderState::garbage_collect()` (or
  the automatic sweep gated by `garbage-collect-states`/
  `garbage-collect-states-rate`) periodically walks the global table and
  frees states with zero references. `clear_cache()` drops all composition
  caches outright; `list_cycles()`/`detect_and_break_cycles()` guard against
  reference cycles that `auto-break-cycles` would otherwise need to catch.
- `get_bin_index()`/`get_draw_order()` lazily inspect the state for a
  `CullBinAttrib` and cache the result (`F_checked_bin_index` flag) — this
  is how the cull pipeline sorts into [CullBin](CullBin.md)s without every
  caller having to know about `CullBinAttrib` specifically.
- `_generated_shader` caches a synthesized `ShaderAttrib` when the state
  contains an "auto" shader attrib (the automatic shader generator's
  output), keyed by `_generated_shader_seq`.
- `MAKE_MAP_PROPERTY(attribs, ...)` exposes the slot contents as a
  dict-like `attribs` property from scripting languages.

## API

| Method | Notes |
|---|---|
| `static CPT(RenderState) make(attrib, override=0)` (1–5 attrib overloads, or `attrib*`/count) | Build a state from scratch with the given attribs, all sharing one override value |
| `static CPT(RenderState) make_empty()` | The identity/empty state |
| `CPT(RenderState) compose(other) const` | Layer `other` on top of `this` (child-relative-to-parent direction), cached |
| `CPT(RenderState) invert_compose(other) const` | Compose with `this`'s inverse; used for computing relative state between two nodes |
| `CPT(RenderState) add_attrib(attrib, override=0) const` | Add/replace, respecting override priority |
| `CPT(RenderState) set_attrib(attrib) const` / `set_attrib(attrib, override) const` | Add/replace unconditionally |
| `CPT(RenderState) remove_attrib(TypeHandle/int slot) const` | Drop one attrib |
| `CPT(RenderState) adjust_all_priorities(int adjustment) const` | Shift every attrib's override by a delta, floored at 0 |
| `bool has_attrib(TypeHandle/int slot) const` / `const RenderAttrib *get_attrib(...) const` | Query one slot |
| `const RenderAttrib *get_attrib_def(int slot) const` | Like `get_attrib` but returns the slot's registered default instead of `nullptr` |
| `int get_override(TypeHandle/int slot) const` | The stored override value for a slot |
| `bool is_empty() const` | No attribs set |
| `int compare_to(other) const` / `compare_sort(other) const` / `compare_mask(other, SlotMask) const` | Ordering comparisons; `compare_mask` restricts to a subset of slots |
| `size_t get_hash() const` | Content hash (lazily computed, cached) |
| `bool has_cull_callback() const` / `bool cull_callback(trav, data) const` | True if any attrib wants a callback during cull; invokes them |
| `int get_bin_index() const` / `int get_draw_order() const` | Cached CullBinAttrib lookups |
| `int get_geom_rendering(int) const` | Adjusts a Geom's rendering-mode bits per the attribs present |
| `static int get_num_states()` / `get_num_unused_states()` | Global interning-table stats |
| `static int garbage_collect()` | Sweep unreferenced interned states now |
| `static int clear_cache()` | Drop all composition-cache entries globally |
| `void output(ostream&) const` / `write(ostream&, int indent) const` | Debug printing |

## Usage

```cpp
CPT(RenderState) s = RenderState::make(
    ColorAttrib::make_flat(LColor(1, 0, 0, 1)),
    TransparencyAttrib::make(TransparencyAttrib::M_alpha));
node_path.set_state(s);

// Force override regardless of what a lower node set:
CPT(RenderState) forced = RenderState::make_empty()
    ->add_attrib(CullFaceAttrib::make(CullFaceAttrib::M_cull_none), 100);
```

## See also

[RenderAttrib](RenderAttrib.md), [RenderAttribRegistry](RenderAttribRegistry.md),
[TransformState](TransformState.md), [AccumulatedAttribs](AccumulatedAttribs.md),
[PandaNode](PandaNode.md), [CullBinManager](CullBinManager.md), module
[README](README.md) ("The state pipeline").
