# TextureContext

**Source:** `panda/src/gobj/textureContext.h` (+ `.I`, `.cxx`)
**Inherits:** [BufferContext](BufferContext.md) > [SavedContext](SavedContext.md); also [AdaptiveLruPage](AdaptiveLru.md)

The GSG-side handle for one uploaded `Texture` view. Created by
`Texture::prepare_now()` via `PreparedGraphicsObjects` (see
[PreparedGraphicsObjects.md](PreparedGraphicsObjects.md) and the README's
"`PreparedGraphicsObjects`/`*Context` handshake" section) and subclassed per
backend (e.g. the GL backend's `CLP(TextureContext)`) to hold the actual API
handle. You touch this class when writing a new `GraphicsStateGuardian`
backend or debugging why a texture isn't re-uploading after a change.

## Behavior notes

- Constructed against a specific `view` index (for multiview/stereo
  textures; `0` in the ordinary case) — one `Texture` can have multiple
  `TextureContext`s registered, one per view, all under the same
  `PreparedGraphicsObjects`.
- Modification tracking is via three independent `UpdateSeq` counters
  (`_properties_modified`, `_image_modified`, `_simple_image_modified`),
  each compared against the live `Texture`'s own sequence counters
  (`get_properties_modified()`/`get_image_modified()`/
  `get_simple_image_modified()`). This lets a GSG backend cheaply check
  *what kind* of re-upload is needed (a full texel re-upload vs. just a
  wrap-mode/filter change) without re-uploading pixel data unnecessarily.
- `mark_loaded()`/`mark_simple_loaded()` snapshot the current `Texture`
  sequence numbers into the context and call `set_resident(true)`
  (`AdaptiveLruPage`); `mark_unloaded()` resets all three counters to
  `UpdateSeq::old()` and marks non-resident — the next `was_modified()`
  check will then always report true, forcing a full re-upload.
  `mark_needs_reload()` only invalidates `_image_modified`, forcing just the
  image (not properties) to be considered stale.
- `update_data_size_bytes()` updates both the `BufferContext` byte-size
  accounting and the `AdaptiveLruPage` eviction-scoring size in one call —
  always call this (not the `BufferContext` one directly) when a backend's
  on-card allocation size changes.
- `get_native_id()`/`get_native_buffer_id()` default to returning `0`;
  backends override them to expose the raw API handle (e.g. a GL texture
  name) for interop/debugging. `get_native_buffer_id()` is specifically for
  buffer textures, which use a separate GPU buffer object identifier.

## API

| Signature | Notes |
|---|---|
| `TextureContext(PreparedGraphicsObjects *pgo, Texture *tex, int view)` | Constructed by `Texture::prepare_now()`; not called directly by app code. |
| `Texture *get_texture() const` | The associated `Texture`. |
| `int get_view() const` | Which view (0 for non-multiview textures). |
| `virtual uint64_t get_native_id() const` / `get_native_buffer_id() const` | Backend-specific raw handle; `0` if unsupported. |
| `bool was_modified() const` | `was_properties_modified() \|\| was_image_modified()`. |
| `bool was_properties_modified/was_image_modified/was_simple_image_modified() const` | Compare context's snapshot vs. the live `Texture`'s current sequence numbers. |
| `UpdateSeq get_properties_modified/get_image_modified/get_simple_image_modified() const` | Raw snapshot accessors. |
| `void mark_loaded()` / `mark_simple_loaded()` | Snapshot current seqs, mark resident. |
| `void mark_unloaded()` | Reset seqs to `old()`, mark non-resident. |
| `void mark_needs_reload()` | Force `was_image_modified()` true next check. |
| `void update_data_size_bytes(size_t)` | Update both `BufferContext` and LRU-page size accounting. |

## See also

- [Texture](Texture.md), [PreparedGraphicsObjects](PreparedGraphicsObjects.md),
  [BufferContext](BufferContext.md), [SavedContext](SavedContext.md),
  [AdaptiveLru](AdaptiveLru.md)
