# SubprocessWindow

**Source:** `panda/src/display/subprocessWindow.h` (+ `.I`, `.cxx`)
**Inherits:** [GraphicsWindow](GraphicsWindow.md) **Inherited by:** (none)

Compiled only when `SUPPORT_SUBPROCESS_WINDOW` is defined — which
`subprocessWindow.h` itself sets automatically only on `IS_OSX` builds, and
`config_display.cxx`'s `init_libdisplay()` only registers the type under the
same guard. On every other platform this class doesn't exist in the build
at all.

Solves a narrow problem: on OSX, a child process cannot draw into, or
attach a window to, a window owned by its parent process. `SubprocessWindow`
renders normally into an internal offscreen `GraphicsBuffer` in the child
process, copies the framebuffer to a shared-memory (`mmap`) region on every
flip, and a parent-process viewer reads it back out. Historically used for
the OSX web-plugin build, where the browser (parent) hosts the visible
window and the Panda app runs as a child process. Not something an ordinary
application constructs — always via `GraphicsEngine::make_output()` with a
`WindowProperties` whose parent-window handle is a
`NativeWindowHandle::SubprocessHandle` (see [WindowHandle.md](WindowHandle.md)).

## Behavior notes

- **Really two windows layered together**: a `SubprocessWindow` itself
  (`GraphicsWindow`) plus an internal `_buffer` (`GraphicsBuffer`) it creates
  and drives via `internal_open_window()`. The `_buffer` is explicitly
  `set_active(false)` so `GraphicsEngine`'s normal per-frame traversal never
  renders it directly — `SubprocessWindow` calls `_buffer->begin_frame()`/
  `end_frame()` itself from its own `begin_frame()`/`end_frame()`
  overrides, i.e. it manually drives a buffer that's otherwise inert to the
  engine.
- **The shared-memory handshake is opened lazily, keyed by filename.**
  `internal_open_window()` pulls the mmap filename from the parent
  `WindowHandle`'s `NativeWindowHandle::SubprocessHandle` (falling back to
  failure if none is set or the filename is empty) and calls
  `SubprocessWindowBuffer::open_buffer()`. If `set_properties_now()` later
  receives a *different* filename than the one currently open, it treats
  that as "re-open the whole window" — full `internal_close_window()` +
  `internal_open_window()` cycle — rather than trying to migrate state.
- **`begin_flip()` does the actual cross-process copy**, and it's the one
  place with real backpressure handling: it copies the rendered frame from
  GPU memory to the `Texture`'s RAM image
  (`_gsg->framebuffer_copy_to_ram(...)`), then waits (spin-polling
  `Thread::force_yield()`) for `_swbuffer->ready_for_write()` — i.e. for the
  parent process to have consumed the *previous* frame — up to
  `subprocess_window_max_wait` seconds (config var, default `0.2`; see
  [README.md](README.md)) before giving up on this frame entirely (silently
  drops it rather than blocking indefinitely) so a hidden/minimized parent
  window can't stall the child's frame loop.
- **`process_events()` decodes a raw input-event queue from shared
  memory.** Each `SubprocessWindowBuffer::Event` carries a source
  (mouse/keyboard), a type (button up/down), an OS-specific keycode, mouse
  position, and a modifier-key bitmask; `SubprocessWindow` diffs the
  modifier bitmask against `_last_event_flags` to synthesize individual
  shift/control/alt/meta up/down transitions (since the shared protocol only
  reports current-state flags, not discrete modifier events), and calls
  `translate_key()` to map OSX-specific virtual keycodes to Panda
  `ButtonHandle`s (a large hardcoded `switch` over Mac scancodes, `#ifdef __APPLE__`
  only — falls back to `ascii_key(os_code & 0xff)` for anything unmapped).
- **Destructor asserts cleanup already happened**: `~SubprocessWindow()`
  `nassertv`s that both `_buffer` and `_swbuffer` are already `nullptr` —
  callers are expected to have gone through `close_window()` (which calls
  `internal_close_window()`) before destruction, not to rely on the
  destructor to tear things down.

## SubprocessWindowBuffer (internal helper, `subprocessWindowBuffer.h/.I/.cxx`)

A plain (non-Panda, non-ref-counted, no `EXPCL` export macro) class that
implements the actual shared-memory protocol, deliberately kept
independent of the rest of Panda so it can be compiled and linked into
code that doesn't otherwise depend on Panda (explicitly called out in the
class comment — used by the Panda3D browser-plugin core API). Placement-
`new`'d directly inside an `mmap`'d region; a fixed-size header
(magic number, `_mmap_size`, dimensions, event ring buffer, read/write
sequence counters) is immediately followed in memory by the raw framebuffer
pixel data (`get_row_size()`/`get_framebuffer_size()`; row size is always
`x_size * 4`, i.e. fixed RGBA8).

- **Two independent single-producer/single-consumer protocols share one
  buffer**: the framebuffer image (producer: child/`SubprocessWindow`,
  consumer: parent) uses a simple double-flag handshake —
  `ready_for_write()` is `_last_written == _last_read`, `ready_for_read()`
  is `_last_written != _last_read`, and `close_write_framebuffer()` just
  increments `_last_written` — while the input-event queue (producer:
  parent, consumer: `SubprocessWindow::process_events()`) is a genuine
  circular buffer of up to `max_events` (64) `Event`s, full when
  `(_event_in + 1) % max_events == _event_out`.
- **Lifecycle is split into "create" (parent) and "attach" (child) pairs**:
  `new_buffer()`/`destroy_buffer()` create the backing temp file (`tmpnam`),
  size it, `mmap` it, and construct the object in place — called by the
  parent process. `open_buffer()`/`close_buffer()` instead `mmap` an
  *existing* file by name (verifying `verify_magic_number()` first, and
  mapping twice — once to read just the header and learn the true
  `_mmap_size`, then again at full size) — called by the child
  (`SubprocessWindow::internal_open_window()`). `close_buffer()` only
  unmaps and closes the fd; it never touches the shared object itself,
  since the other process (or the original creator) may still be using it.
- **The magic number (`"pNdaSWB"`) exists specifically to reject
  accidentally mmap-ing the wrong file** as a `SubprocessWindowBuffer` —
  checked in `open_buffer()` before trusting anything else in the header.

## API (SubprocessWindow overrides)

| Signature | Notes |
|---|---|
| `virtual void process_events()` | Calls `GraphicsWindow::process_events()` first, then drains the shared-memory event queue. |
| `virtual bool begin_frame(FrameMode, Thread*)` | Delegates to the internal `_buffer`. |
| `virtual void end_frame(FrameMode, Thread*)` | Delegates to `_buffer`; sets `_flip_ready` on `FM_render`. |
| `virtual void begin_flip()` | Copies the framebuffer to the shared-memory region; see behavior notes. |
| `virtual void set_properties_now(WindowProperties&)` | Detects a changed mmap filename and triggers a full re-open. |
| `virtual void close_window()` (protected) | → `internal_close_window()` + throws the window-closed event via `system_changed_properties()`. |
| `virtual bool open_window()` (protected) | → `internal_open_window()` + throws the window-opened event. |

## See also

- [GraphicsWindow.md](GraphicsWindow.md) — base class.
- [GraphicsBuffer.md](GraphicsBuffer.md) — the internal offscreen buffer this class drives manually.
- [WindowHandle.md](WindowHandle.md) — `NativeWindowHandle::SubprocessHandle`, the parent-window handle type that carries the mmap filename.
- [README.md](README.md) — `subprocess-window` / `subprocess-window-max-wait` config variables.
