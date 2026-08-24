# SavedContext

**Source:** `panda/src/gobj/savedContext.h` (+ `.I`, `.cxx`)
**Inherits:** TypedObject **Inherited by:** [BufferContext](BufferContext.md), [GeomContext](GeomContext.md), [SamplerContext](SamplerContext.md), [ShaderContext](ShaderContext.md)

Abstract base for every GSG-specific "handle to an uploaded resource" class — see the module README's [`PreparedGraphicsObjects`/`*Context` handshake](README.md#preparedgraphicsobjects--context-handshake) for the overall pattern this participates in. `SavedContext` itself carries **no state at all**; it exists purely to give the `*Context` family a common base type so [`PreparedGraphicsObjects`](PreparedGraphicsObjects.md) can hold/manage them polymorphically, and to provide a uniform `output()`/`write()` debug-print interface. All of the actual per-resource bookkeeping (size in bytes, modified sequence, residency) lives in the more specific subclasses, chiefly [`BufferContext`](BufferContext.md).

## Behavior notes

- The default constructor and both virtual methods are trivial — `output()` just prints `"SavedContext " << this`, and `write()` indents and calls `output()`. Every subclass overrides `output()` to say something more useful; if the plain `"SavedContext 0x..."` output appears, a subclass has forgotten to override `output()`.
- Because it carries no data, there's no `release`/cleanup logic here — subclasses own that entirely.

## API

| Signature | Notes |
|---|---|
| `virtual void output(std::ostream &out) const` | Prints a one-line description; overridden by every real subclass. |
| `virtual void write(std::ostream &out, int indent_level) const` | Indents, then calls `output()`. |
| `static TypeHandle get_class_type()` | Standard Panda RTTI accessor. |

## Usage

`SavedContext` is never constructed directly — always through a concrete subclass, typically returned from a `prepare_now()` call:

```cpp
TextureContext *tc = my_texture->prepare_now(prepared_objects, gsg);
// tc is-a SavedContext; polymorphic code can print it generically:
tc->write(std::cout, 0);
```

## See also

- [BufferContext](BufferContext.md), [GeomContext](GeomContext.md), [SamplerContext](SamplerContext.md), [ShaderContext](ShaderContext.md) — the concrete subclasses
- [PreparedGraphicsObjects](PreparedGraphicsObjects.md) — owns and dispatches all `*Context` objects for one GSG
