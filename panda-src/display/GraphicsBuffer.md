# GraphicsBuffer

**Source:** `panda/src/display/graphicsBuffer.h` (+ `.I`, `.cxx`)
**Inherits:** [GraphicsOutput](GraphicsOutput.md) **Inherited by:** (per-GSG-backend concrete offscreen buffer classes, outside this module)

An offscreen render target — functionally like a `GraphicsWindow` but with
no visible output. Adds almost nothing to `GraphicsOutput` at this base
level: a resizability check, and the open/close request state machine
(`OR_none`/`OR_open`/`OR_close`) that lets `GraphicsEngine` defer actually
opening or closing the buffer until the next `process_events()` call on the
window thread. Constructor is `protected` — created via
`GraphicsEngine::make_output()` (referred to in the source comment as
`make_buffer()`, an older name for the same entry point). The real
graphics-context work (`open_buffer()`/`close_buffer()`) is deliberately
trivial here (`open_buffer()` just returns `false`) — concrete
implementations live in each GSG backend module (GL, DX, etc.), not in
`panda/src/display`.

## Behavior notes

- **`set_size()` refuses to resize unless the buffer was created with the
  `GraphicsPipe::BF_resizeable` flag** — raises an assertion
  (`nassert_raise`) and returns without effect otherwise. Most
  render-to-texture buffers created via `make_texture_buffer()` are *not*
  resizeable by default.
- **`process_events()` snapshots and clears `_open_request` before acting
  on it**, specifically to tolerate a recursive call back into
  `process_events()` triggered by the open/close logic itself — if it
  didn't reset the flag first, a recursive re-entry could see and re-act on
  a stale request.
- **A successful open also fixes up the inverted flag**: after
  `open_buffer()` succeeds, `process_events()` calls
  `set_inverted(_gsg->get_copy_texture_inverted())` — so whether the buffer
  renders right-side-up or flipped is determined by the GSG's own
  framebuffer-to-texture copy convention, not by the `window-inverted`
  config default that a `GraphicsWindow` would otherwise pick up (see
  `PandaFramework`/`WindowFramework` framework-module docs for the
  window-level default).
- **The base `close_buffer()` only logs**; `open_buffer()` only returns
  `false` — both are meant to be overridden by concrete backend subclasses.

## API

| Signature | Notes |
|---|---|
| `GraphicsBuffer(GraphicsEngine*, GraphicsPipe*, const string &name, const FrameBufferProperties&, const WindowProperties&, int flags, GraphicsStateGuardian*, GraphicsOutput *host)` (protected) | Passes `default_stereo_flags=false` up to `GraphicsOutput`. |
| `virtual ~GraphicsBuffer()` | |
| `virtual void set_size(int x, int y)` | Requires `BF_resizeable`; delegates to `GraphicsOutput::set_size_and_recalc()`. |
| `virtual void request_open()` / `virtual void request_close()` | Sets `_open_request`; actual work deferred to `process_events()`. |
| `virtual void set_close_now()` | Immediate synchronous close (window thread only). |
| `virtual void process_events()` | Honors `_open_request`; see behavior notes. |
| `virtual void close_buffer()` (protected) | Base: logs only. |
| `virtual bool open_buffer()` (protected) | Base: returns `false`. |
| `enum OpenRequest { OR_none, OR_open, OR_close }` (protected) | |

## See also

- [GraphicsOutput.md](GraphicsOutput.md) — base class; most of a `GraphicsBuffer`'s useful API lives there.
- [ParasiteBuffer.md](ParasiteBuffer.md) — an alternative offscreen strategy that shares another output's framebuffer instead of allocating its own.
- [GraphicsEngine.md](GraphicsEngine.md) — creates `GraphicsBuffer` instances.
- [README.md](README.md) — `prefer-texture-buffer`/`prefer-parasite-buffer`/`force-parasite-buffer` config vars that steer whether a `GraphicsOutput::make_texture_buffer()` call produces a `GraphicsBuffer` or a `ParasiteBuffer`.
