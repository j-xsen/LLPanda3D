# StateMunger

**Source:** `panda/src/pgraph/stateMunger.h` (+ `.I`, `.cxx`)
**Inherits:** GeomMunger (see `gobj`, undocumented)

A thin `GeomMunger` specialization that adds the ability to munge a
[RenderState](RenderState.md), not just a Geom's vertex data — `GeomMunger`
itself can't know about `RenderState` (it would create a circular
dependency between `gobj` and `pgraph`), so this subclass exists purely to
bridge the two. A `GraphicsStateGuardian` implementation typically
subclasses `StateMunger` (rather than `GeomMunger` directly) whenever it
needs GSG-specific state adjustments — e.g. synthesizing/rewriting a
`ShaderAttrib`, or normalizing texture/color attribs the hardware can't
represent directly.

## Behavior notes

- **Per-state, per-GSG memoization**: `munge_state()` doesn't recompute on
  every call — it checks `state->_munged_states` (a private cache living on
  the `RenderState` itself, keyed by GSG id) for an existing weak-pointer
  result and returns it if still alive; otherwise it calls the virtual
  `munge_state_impl()` and stores the result back into that same
  `RenderState`-owned cache. This is why `RenderState::clear_munger_cache()`
  exists — it's the sanctioned way to invalidate every state's cached
  munge results at once (e.g. after a GSG capability change).
- The default `munge_state_impl()` is a no-op (`return state;`) — a
  concrete GSG subclass overrides it to actually do work; `StateMunger`
  itself never rewrites a state.
- `should_munge_state()` is a cheap precomputed flag (`_should_munge_state`,
  set by the constructor/subclass) letting callers skip calling
  `munge_state()` entirely for GSGs that never need to.

## API

| Method | Notes |
|---|---|
| `StateMunger(GraphicsStateGuardianBase *gsg)` | Construct bound to one GSG |
| `CPT(RenderState) munge_state(const RenderState *state)` | Cached entry point |
| `bool should_munge_state() const` | Fast check: does this GSG need munging at all? |
| `virtual CPT(RenderState) munge_state_impl(const RenderState *state)` | Override point for concrete GSGs; default is identity |

## See also

[RenderState](RenderState.md), `display`'s
[GraphicsStateGuardian](../display/GraphicsStateGuardian.md).
