# WindowHandle

**Source:** `panda/src/display/windowHandle.h` (+ `.I`, `.cxx`)
**Inherits:** TypedReferenceCount **Inherited by:** `NativeWindowHandle` (folded into this doc, below)

An opaque, OS-agnostic wrapper around a native desktop window handle —
*not necessarily a Panda window*. Assigned to
[WindowProperties](WindowProperties.md)'s `parent_window` property to
embed a Panda window inside another window (e.g. a browser plugin, or a
host application's UI), and used bidirectionally: the parent's
`WindowHandle` can deliver OS messages down to the child, and the child can
request keyboard focus from the parent.

## Behavior notes

- **Two-layer design.** `WindowHandle` itself holds only a `PT(OSHandle)`
  — a nested abstract class (`WindowHandle::OSHandle`, itself
  `TypedReferenceCount`) whose base implementation is a stub: default
  `get_int_handle()` returns `0`, default `output()` prints `"(no type)"`.
  All the real platform-specific storage lives in `OSHandle` subclasses
  defined by `NativeWindowHandle` (below). This split is what lets
  `WindowHandle` stay a single non-templated type usable uniformly by
  `WindowProperties`/`GraphicsWindow` regardless of platform.
- **A `WindowHandle` is never constructed directly in normal code** — the
  header explicitly calls for one of `NativeWindowHandle`'s `make_*()`
  factory methods instead; `NativeWindowHandle`'s own constructors are
  `private`, and it exists (per its header comment) "for name scoping
  only."
- **Parent/child keyboard-focus protocol.** `attach_child()`/
  `detach_child()` are called (by the embedding machinery) to register/
  unregister a child `WindowHandle` with a parent; `request_keyboard_focus(child)`
  records which child is the current keyboard target
  (`_keyboard_window`); `send_windows_message(msg, wparam, lparam)`, called
  on the *parent*, forwards a Windows message to whichever child currently
  holds `_keyboard_window` by calling that child's
  `receive_windows_message()`. The base `receive_windows_message()`
  implementation just logs the message to `nout` — a real embedding
  scenario (e.g. the web plugin system) is expected to override it. This
  whole mechanism exists specifically to route input on Windows Vista+,
  where the browser detects button events directly and must hand them to
  the embedded Panda window explicitly (per the `.cxx` comment).
- **`get_int_handle()`** delegates to the underlying `OSHandle`'s override
  and returns `0` if there's no handle or the representation can't be
  expressed as an integer (e.g. `SubprocessHandle`, below, has no override
  and inherits the base `0`).

## NativeWindowHandle (subclass, `final`)

The one concrete `WindowHandle` implementation, providing a factory method
per kind of native handle plus a matching `OSHandle` subclass to store it:

| Factory | Compiled when | Underlying `OSHandle` | Stores |
|---|---|---|---|
| `static PT(WindowHandle) make_int(size_t window)` | always | `IntHandle` | A raw integer, understood as an `HWND` or X11 `Window` cast to an integer. Documented as primarily for Python's convenience — C++ callers should prefer `make_x11()`/`make_win()`. |
| `static PT(WindowHandle) make_subprocess(const Filename &filename)` | always | `SubprocessHandle` | The mmap/pipe filename of a [SubprocessWindow](SubprocessWindow.md)'s `SubprocessWindowBuffer`, running in another process. Per the `.cxx` comment, useful mainly (perhaps only) on macOS, where parenting child windows is otherwise particularly problematic. |
| `static PT(WindowHandle) make_x11(X11_Window window)` | `HAVE_X11 && !CPPPARSER` | `X11Handle` | A real X11 `Window` id. |
| `static PT(WindowHandle) make_win(HWND window)` | `WIN32 && !CPPPARSER` | `WinHandle` | A real Win32 `HWND`. |

Each `OSHandle` subclass overrides `get_int_handle()` (where the
representation is naturally integral — `IntHandle`, `X11Handle`,
`WinHandle`; `SubprocessHandle` does not override it and returns `0`) and
`output()` for diagnostics. `NativeWindowHandle::init_type()` registers all
of these nested handle types with the `TypeRegistry` alongside itself.

This is the class that `config_display.h`'s `parent-window-handle` and
`subprocess-window` variables get turned into (see
[WindowProperties.md](WindowProperties.md)'s behavior notes on
`get_config_properties()`), via `make_int()` and `make_subprocess()`
respectively.

## API

| Signature | Notes |
|---|---|
| `INLINE WindowHandle(OSHandle *os_handle)` / `INLINE WindowHandle(const WindowHandle &copy)` | Normally called only via `NativeWindowHandle::make_*()`. |
| `virtual ~WindowHandle()` | |
| `INLINE OSHandle *get_os_handle() const` / `INLINE void set_os_handle(OSHandle *)` | |
| `size_t get_int_handle() const` | `0` if unavailable for this handle type. |
| `void output(std::ostream &out) const` | Delegates to the `OSHandle`'s `output()`, or prints `"(null)"`. |
| `virtual void attach_child(WindowHandle *child)` | Base implementation is a no-op — override for real embedding. |
| `virtual void detach_child(WindowHandle *child)` | Clears `_keyboard_window` if it was `child`. |
| `virtual void request_keyboard_focus(WindowHandle *child)` | Sets `_keyboard_window`. |
| `void send_windows_message(unsigned int msg, int wparam, int lparam)` | Call on the *parent*; forwards to the current `_keyboard_window` child's `receive_windows_message()`. |
| `virtual void receive_windows_message(unsigned int msg, int wparam, int lparam)` | Call on a *child*; base implementation just logs to `nout`. |

## Usage

```cpp
#include "nativeWindowHandle.h"
#include "windowProperties.h"

// Embed a Panda window inside an existing native window (e.g. HWND hwnd):
PT(WindowHandle) parent = NativeWindowHandle::make_win(hwnd);

WindowProperties props;
props.set_parent_window(parent);
props.set_origin(0, 0);
props.set_size(800, 600);
// ... pass props to GraphicsEngine::make_output() / GraphicsWindow::request_properties()
```

## See also

- [WindowProperties.md](WindowProperties.md) — `parent_window` property carries a `WindowHandle*`; `get_config_properties()` builds one from config vars.
- [GraphicsPipe.md](GraphicsPipe.md) — `make_window_handle()`, the platform-`GraphicsPipe`-specific alternative entry point mentioned in `WindowProperties::set_parent_window()`'s docs.
- [SubprocessWindow.md](SubprocessWindow.md) — the counterpart consumer of `NativeWindowHandle::make_subprocess()`'s handle.
- [GraphicsWindow.md](GraphicsWindow.md) — the window that actually gets embedded via a `WindowHandle`.
