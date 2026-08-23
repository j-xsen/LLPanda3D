# GeomContext

**Source:** `panda/src/gobj/geomContext.h` (+ `.I`, `.cxx`)
**Inherits:** [SavedContext](SavedContext.md) **Inherited by:** (none)

GSG-side handle for a [`Geom`](Geom.md) prepared in **retained mode** — the header's own example is an OpenGL display-list identifier. This is a legacy/optional rendering path: modern GSG backends primarily render `Geom`s immediate-mode each frame via their `GeomVertexData`/`GeomPrimitive` buffers rather than baking a whole `Geom` into a single opaque replay handle, so `GeomContext` sees far less use than `TextureContext`/`VertexBufferContext`/`IndexBufferContext`.

## Behavior notes

- Extremely thin — the entire class is one raw pointer back to the `Geom` it represents (`_geom`, `public` rather than `private`, unusually for this family) plus the standard `SavedContext` machinery. No size/residency/modified-sequence tracking at all, unlike `BufferContext`'s subclasses — display-list-style caching doesn't have a "bytes on the card" concept in the same way a buffer does.
- Same non-owning-pointer caveat as `BufferContext::_object`: it's a raw `Geom*`, not a `PT(Geom)`, to avoid a reference cycle between the `Geom` and the GSG that both reference each other's context.
- No `.cxx` logic beyond the `TypeHandle` registration — everything is inline.

## API

| Signature | Notes |
|---|---|
| `GeomContext(Geom *geom)` | Wraps the given Geom. |
| `Geom *get_geom() const` | The Geom this context represents. |

## Usage

```cpp
GeomContext *gc = my_geom->prepare_now(prepared_objects, gsg);
// GSG backend stores/replays whatever display-list-style handle it created
```

## See also

- [Geom](Geom.md) — the object this context backs
- [SavedContext](SavedContext.md) — base class
- [PreparedGraphicsObjects](PreparedGraphicsObjects.md) — owns and dispatches these
