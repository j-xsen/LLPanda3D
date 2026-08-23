# TextureReloadRequest

**Source:** `panda/src/gobj/textureReloadRequest.h` (+ `.I`, `.cxx`)
**Inherits:** AsyncTask (see [../event/AsyncTask.md](../event/AsyncTask.md))

A background-thread task that forces a `Texture`'s RAM image to be re-read
from disk (via `get_ram_image()`/`get_uncompressed_ram_image()`), then
re-enqueues it with `PreparedGraphicsObjects`. Created and scheduled by
`GraphicsStateGuardian::async_reload_texture()` when
`get_incomplete_render()` is enabled — app code doesn't normally construct
this directly.

## Behavior notes

- `do_task()` first checks whether a reload is even still needed
  (`_texture->was_image_modified(pgo)` and whether the right kind of RAM
  image — compressed-allowed vs. uncompressed-only — is already present);
  if not, it returns `DS_done` immediately without touching disk. This
  guards against redundant reloads if the texture was already reloaded by
  another path between when this task was queued and when it actually runs.
- If `async-load-delay` (config var, [README](README.md#config-variables-from-config_gobjh))
  is nonzero, the task sleeps that long (simulating slow I/O for testing)
  and **re-checks** the same "still needed" condition before actually
  loading — a concurrent reload during the sleep can still short-circuit
  the disk read.
- After loading, it always calls `_pgo->enqueue_texture(_texture)` even
  though the task always returns `DS_done` (never reschedules itself) —
  this is explicitly to prevent a leak: without re-enqueuing, a texture
  reloaded into RAM but not currently visible in any frame would never get
  prepared onto the GPU, and Panda would just hold the freshly-loaded RAM
  image forever instead of it eventually being evicted once actually
  prepared/released.
- `_allow_compressed` selects `get_ram_image()` (may return compressed
  data as-is) vs. `get_uncompressed_ram_image()` (always decompresses) —
  matches whichever the requesting GSG can consume directly.

## API

| Signature | Notes |
|---|---|
| `TextureReloadRequest(const string &name, PreparedGraphicsObjects *pgo, Texture *texture, bool allow_compressed)` | |
| `PreparedGraphicsObjects *get_prepared_graphics_objects() const` | |
| `Texture *get_texture() const` | Also `MAKE_PROPERTY(texture, ...)`. |
| `bool get_allow_compressed() const` | |
| `bool is_ready() const` | Inherited-pattern readiness check (task-specific; see `AsyncTask`). |

## See also

- [Texture](Texture.md), [PreparedGraphicsObjects](PreparedGraphicsObjects.md),
  [../event/AsyncTask.md](../event/AsyncTask.md)
