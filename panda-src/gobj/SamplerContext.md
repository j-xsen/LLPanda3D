# SamplerContext

**Source:** `panda/src/gobj/samplerContext.h` (+ `.I`, `.cxx`)
**Inherits:** [SavedContext](SavedContext.md), SimpleLruPage **Inherited by:** GSG-backend-specific subclasses (not part of `gobj`)

The GSG-side handle for one uploaded [SamplerState](SamplerState.md) —
same "prepared object" pattern as `TextureContext`/`ShaderContext` (see
the module README's "`PreparedGraphicsObjects` / `*Context` handshake").
Exists because some backends (OpenGL) have mutable sampler objects while
others (Direct3D 10+) have immutable ones bound at creation; giving every
*unique* `SamplerState` value its own `SamplerContext` sidesteps that
difference and guarantees no duplicate sampler objects exist per GSG for
the same settings — `sampler-object-limit` (config var, see module
README) caps how many are kept live.

## Behavior notes

- Inherits `SimpleLruPage` (not `AdaptiveLruPage` like the buffer-family
  contexts) — sampler objects are tracked on a plain LRU, not the
  cost-weighted `AdaptiveLru` used for texture/vertex/index buffer GPU
  residency; see the module README's "Residency tracking" section.
- The base-class constructor takes a `const SamplerState &sampler`
  parameter but doesn't store it anywhere — neither `SamplerContext` nor
  `SavedContext` keeps a copy. A concrete backend subclass is expected to
  capture whatever it needs (the driver-specific sampler object handle)
  itself; this base class is a pure identity/LRU-tracking shell.
- `output()`/`write()` just delegate to `SavedContext`'s — no extra state
  to print at this level.

## API

| Signature | Notes |
|---|---|
| `SamplerContext(const SamplerState &sampler)` | Registers as an `SimpleLruPage(1)`; see note above re: the unused parameter. |
| `virtual void output(ostream &) const` / `virtual void write(ostream &, int indent_level) const` | Debug printing, delegates to `SavedContext`. |

## See also

- [SamplerState](SamplerState.md) — the value type this is a GPU-side
  handle for
- [SavedContext](SavedContext.md), [PreparedGraphicsObjects](PreparedGraphicsObjects.md)
- [SimpleLru](SimpleLru.md) — the LRU this class is a page of
