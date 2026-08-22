# ParasiteBuffer

**Source:** `panda/src/display/parasiteBuffer.h` (+ `.I`, `.cxx`)
**Inherits:** [GraphicsOutput](GraphicsOutput.md) directly — **not** [GraphicsBuffer](GraphicsBuffer.md) **Inherited by:** (none)

A `GraphicsOutput` that behaves like an offscreen buffer but allocates no
framebuffer memory of its own — instead it renders into the framebuffer
already owned by some other `GraphicsOutput` (its `host`), and relies on the
caller copying that framebuffer content to a texture immediately afterward.
Exists for backends that either can't render directly into a texture (must
render to a separate framebuffer first, then copy) or can't do offscreen
rendering at all — a `ParasiteBuffer` gives them a way to produce a texture
without wasting a whole extra framebuffer's worth of graphics memory.
`has_texture()` is implicitly assumed true for any `ParasiteBuffer` — the
whole class only makes sense when render-to-texture is actually in use.
Unlike every other `GraphicsOutput` subclass, its constructor is **public**,
not protected, but the class doc still directs callers through
`GraphicsEngine::make_parasite()` rather than constructing directly.

## Behavior notes

- **The buffer's size must fit within the host's bounds.** `begin_frame()`
  shrinks the parasite's own size (via `set_size_and_recalc()`) down to the
  host's current size whenever the host is smaller — either by tracking the
  host exactly (if created with `GraphicsPipe::BF_size_track_host`) or by
  clamping to `min(own size, host size)` otherwise. It never grows itself
  back up automatically once the host shrinks and grows again unless
  `BF_size_track_host` was set.
- **`set_size_and_recalc()` applies power-of-2/square constraints only
  when *not* tracking the host** — `BF_size_power_2` and `BF_size_square`
  creation flags are ignored while `BF_size_track_host` is in effect (since
  then the size is dictated entirely by the host, not by these flags).
- **Almost every draw/flip lifecycle method is forwarded straight to the
  host**: `flip_ready()`, `begin_flip()`, `ready_flip()`, `end_flip()` all
  just call the same method on `_host`. `begin_frame()` calls
  `_host->begin_frame(FM_parasite, ...)` (note the mode is always
  `FM_parasite`, regardless of what mode was requested on the parasite
  itself) — meaning the host actually does the real rendering setup; the
  parasite is a thin view into that same render.
- **`end_frame()` is where the actual "copy to texture" happens** — after
  forwarding to `_host->end_frame(FM_parasite, ...)`, if `mode == FM_render`
  it calls `promote_to_copy_texture()` (demoting any `RTM_bind_or_copy`
  texture request down to a real copy, since a `ParasiteBuffer` can never
  bind directly) followed by `copy_to_textures()` (inherited from
  `GraphicsOutput`). `FM_refresh` mode skips this entirely and just returns.
- **`is_active()` requires both itself *and* its host to be active** —
  `GraphicsOutput::is_active() && _host->is_active()` — so a parasite buffer
  goes inactive automatically whenever its host does, with no separate
  bookkeeping needed.
- **`get_host()` returns `_host`, not `this`** — the one meaningful override
  of `GraphicsOutput::get_host()` (which returns `this` at the base level).
  This is what lets `GraphicsOutput::make_texture_buffer()`'s "normally
  the buffer would be its own host" logic correctly attribute a chain of
  nested parasite-on-parasite creation back to the real underlying output.
- **Construction immediately marks itself valid** (`_is_valid = true`) and
  computes its inverted flag from the host's GSG
  (`host->get_gsg()->get_copy_texture_inverted()`) — there is no separate
  open/close request state machine like `GraphicsBuffer` has; a
  `ParasiteBuffer` is ready to use as soon as it's constructed, since it
  never owns any real graphics-context resources to open.

## API

| Signature | Notes |
|---|---|
| `ParasiteBuffer(GraphicsOutput *host, const string &name, int x_size, int y_size, int flags)` (public) | Prefer `GraphicsEngine::make_parasite()` in practice. |
| `virtual ~ParasiteBuffer()` | |
| `virtual bool is_active() const` | AND's with the host's active state. |
| `void set_size(int x, int y)` | Requires `BF_resizeable`. |
| `void set_size_and_recalc(int x, int y)` | Applies power-of-2/square flags only when not host-tracking. |
| `virtual bool flip_ready() const` / `virtual void begin_flip()` / `virtual void ready_flip()` / `virtual void end_flip()` | All forward to `_host`. |
| `virtual bool begin_frame(FrameMode mode, Thread *)` / `virtual void end_frame(FrameMode mode, Thread *)` | See behavior notes — host does the real rendering, parasite handles the texture copy. |
| `virtual GraphicsOutput *get_host()` | Returns `_host`. |

## See also

- [GraphicsOutput.md](GraphicsOutput.md) — base class; `make_texture_buffer()` there is the usual way a `ParasiteBuffer` gets created indirectly.
- [GraphicsBuffer.md](GraphicsBuffer.md) — the alternative "own framebuffer" offscreen strategy.
- [GraphicsPipe.md](GraphicsPipe.md) — `BF_size_track_host`/`BF_size_power_2`/`BF_size_square`/`BF_resizeable` creation flags.
- [README.md](README.md) — `prefer-parasite-buffer`/`force-parasite-buffer` config vars that steer `make_texture_buffer()` toward creating one of these.
