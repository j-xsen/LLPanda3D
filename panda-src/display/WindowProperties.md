# WindowProperties

**Source:** `panda/src/display/windowProperties.h` (+ `.I`, `.cxx`)
**Inherits:** (none — standalone value class) **Inherited by:** (none)

A property bag describing the configuration of a graphics window: origin,
size, title, decoration/fixed-size/fullscreen/foreground/minimized/
cursor-hidden flags, icon/cursor filenames, z-order, mouse mode, and parent
window (for embedding). Used both to *request* a configuration — passed to
`GraphicsEngine::make_output()` or [GraphicsWindow](GraphicsWindow.md)'s
`request_properties()` — and, after a window is open, to report its
*current* configuration back (`GraphicsWindow::get_properties()`).

## Behavior notes

- **Every property is independently "specified" or not**, via a private
  `_specified` bitmask (`S_origin`, `S_size`, `S_title`, ... one bit per
  property) checked by `has_X()` and set by `set_X()`/cleared by
  `clear_X()`. This is the whole point of the class: a `WindowProperties`
  passed to `request_properties()` only touches the fields it has
  explicitly specified — leaving the rest of the window's current
  configuration untouched. `is_any_specified()` reports whether *anything*
  has been set. `get_X()` on an unspecified field asserts (`nassertr`) and
  returns a zero/default value rather than raising — so callers must guard
  with `has_X()` first, not rely on a sentinel return value.
- **Boolean properties store their value in a second, separate bitmask**
  (`_flags`, using `F_*` bits that numerically alias the corresponding
  `S_*` bits) — `_specified` says *whether* a bool property was set,
  `_flags` says *what* it was set to. `clear_X()` unsets both bits.
- **`add_properties(other)` is the merge operation**, used to layer one
  `WindowProperties` on top of another — it copies over only the fields
  `other.has_X()` reports as specified, leaving everything else on `this`
  alone. This is what `WindowFramework`/`GraphicsEngine` use to combine
  explicit caller properties with `get_default()`.
- **`get_default()` / `set_default()` / `clear_default()`** let an
  application override the process-wide default `WindowProperties`
  (returned by `get_default()`) with a custom one; without an override,
  `get_default()` falls through to `get_config_properties()`, which builds
  a fresh `WindowProperties` from the `config_display.h` variables already
  tabulated in [README.md](README.md) (`win-size`, `win-origin`,
  `fullscreen`, `undecorated`, `win-fixed-size`, `cursor-hidden`,
  `icon-filename`, `cursor-filename`, `z-order`, `window-title`,
  `parent-window-handle`/`subprocess-window`). `get_config_properties()`
  always sets `open` true and `mouse_mode` to `M_absolute`.
- **`parent_window_handle`/`subprocess_window` config vars become a real
  `WindowHandle`** at `get_config_properties()` time via
  `NativeWindowHandle::make_int()` or `NativeWindowHandle::make_subprocess()`
  — see [WindowHandle.md](WindowHandle.md).
- **The deprecated `set_parent_window(size_t)` overload** exists only for
  backward compatibility and interprets the raw integer as a
  platform-specific handle value (an `HWND` on Windows, an X11 `Window` id,
  or — noted in the source as not actually working — an `NSWindow*` on
  OSX); it wraps the value with `NativeWindowHandle::make_int()` and
  delegates to the `WindowHandle*` overload. New code should construct a
  `WindowHandle` explicitly via `GraphicsPipe::make_window_handle()`
  instead.
- **The static factory methods `size(...)`** build a `WindowProperties`
  with only the size field specified — documented as deprecated in the
  Python API (`WindowProperties(size=(x, y))` preferred there), but still
  the normal C++ idiom for buffer-only size requests, since size is the
  only property that matters to an offscreen buffer.
- **`M_relative` and `M_confined` mouse modes are platform-limited**:
  `M_relative` has no effect on Windows at all, and on Unix/X11 requires
  the `Xxf86dga` extension; `M_confined` (absolute coordinates, pointer
  clamped to the window) is the portable substitute for FPS-style mouse
  capture.
- **`operator==`/`operator!=` compare every field including `_specified`
  and `_flags` directly** — two `WindowProperties` are equal only if they
  have the *same set* of specified fields with the *same* values, not just
  equivalent effective configuration.

## API

### Construction / global defaults

| Signature | Notes |
|---|---|
| `WindowProperties()` | All fields unspecified (`clear()`). |
| `INLINE WindowProperties(const WindowProperties &copy)` | |
| `void operator = (const WindowProperties &copy)` | |
| `static WindowProperties get_config_properties()` | Built fresh from `config_display.h` vars each call. |
| `static WindowProperties get_default()` | Returns the app-level override if `set_default()` was called, else `get_config_properties()`. |
| `static void set_default(const WindowProperties &)` | Replaces (not merges) the default. |
| `static void clear_default()` | Reverts to config-file-derived defaults. |
| `static WindowProperties size(const LVecBase2i &)` / `size(int, int)` | Size-only properties; primarily for buffers. |
| `void clear()` | Resets to fully-unspecified. |
| `INLINE bool is_any_specified() const` | |
| `void add_properties(const WindowProperties &other)` | Merges only `other`'s specified fields onto `this`. |
| `bool operator == / != (const WindowProperties &) const` | Exact field-for-field comparison, see behavior notes. |
| `void output(std::ostream &out) const` | Human-readable dump of only the specified fields. |

### Per-property accessors (all follow the `set_X`/`get_X`/`has_X`/`clear_X` pattern)

| Property | Type | Notes |
|---|---|---|
| `origin` | `LPoint2i` (+ `get_x_origin()`/`get_y_origin()`) | Top-left corner in screen pixels, excluding decorations. |
| `size` | `LVector2i` (+ `get_x_size()`/`get_y_size()`) | Excludes decorations. |
| `title` | `std::string` | |
| `undecorated` | `bool` | No title bar/border when true. |
| `fixed_size` | `bool` | Not user-resizable when true. |
| `fullscreen` | `bool` | |
| `foreground` | `bool` | |
| `minimized` | `bool` | |
| `raw_mice` | `bool` | Read raw mouse devices (no `MAKE_PROPERTY2`, plain accessors only). |
| `open` | `bool` | Can construct a `GraphicsWindow` closed, then request open later. |
| `cursor_hidden` | `bool` | |
| `icon_filename` | `Filename` | Window/taskbar icon. |
| `cursor_filename` | `Filename` | Custom mouse cursor image. |
| `z_order` | `ZOrder` (`Z_bottom`/`Z_normal`/`Z_top`) | |
| `mouse_mode` | `MouseMode` (`M_absolute`/`M_relative`/`M_confined`) | See behavior notes on platform limits. |
| `parent_window` | `WindowHandle*` | For embedding; see [WindowHandle.md](WindowHandle.md). Also: deprecated `set_parent_window(size_t)`. |

`ZOrder` and `MouseMode` both have `operator<<`/`operator>>` for
stream I/O (used by `output()` and by config-var parsing of `z-order`).

## Usage

```cpp
#include "windowProperties.h"

WindowProperties props = WindowProperties::get_default();
props.set_title("My App");
props.set_size(1280, 720);
props.set_fullscreen(false);
props.set_undecorated(false);

// Later, to change just one thing on an already-open window:
WindowProperties resize;
resize.set_size(1920, 1080);
window->request_properties(resize);  // GraphicsWindow — leaves title, etc. untouched
```

## See also

- [WindowHandle.md](WindowHandle.md) — `parent_window` property type; also created from `parent-window-handle`/`subprocess-window` config vars.
- [GraphicsWindow.md](GraphicsWindow.md) — `request_properties()`/`get_properties()` consume/produce these.
- [GraphicsPipe.md](GraphicsPipe.md) — `make_window_handle()`, the recommended way to build a `WindowHandle` for `parent_window`.
- [README.md](README.md) — the underlying `config_display.h` variables (`win-size`, `win-origin`, `fullscreen`, etc.) read by `get_config_properties()`.
- [../framework/PandaFramework.md](../framework/PandaFramework.md) — `get_default_window_props()` builds on `WindowProperties::get_default()`.
