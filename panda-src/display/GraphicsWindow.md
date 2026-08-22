# GraphicsWindow

**Source:** `panda/src/display/graphicsWindow.h` (+ `.I`, `.cxx`)
**Inherits:** [GraphicsOutput](GraphicsOutput.md) **Inherited by:** [CallbackGraphicsWindow](CallbackGraphicsWindow.md), [SubprocessWindow](SubprocessWindow.md), and the platform-specific window classes in each API's module (`glxGraphicsWindow`, `wglGraphicsWindow`, etc.)

A `GraphicsOutput` that is an actual OS window — fullscreen or on the
desktop. Adds to `GraphicsOutput`: the input-device list (keyboard/mouse,
see [GraphicsWindowInputDevice](GraphicsWindowInputDevice.md)),
`WindowProperties`-based open/resize/close request plumbing, the raw
OS-message callback hook system (`GraphicsWindowProc`), and window-level
events (`window-event`, close-request event). Like `WindowFramework`
(`framework` module), the constructor is `protected` — real instances only
come from `GraphicsEngine::make_output()`, which picks the concrete
platform subclass.

## Behavior notes

- **The properties state machine is deliberately asynchronous and
  thread-separated.** `request_properties()` only merges the request into
  `_requested_properties` under `_properties_lock` — it does **not** apply
  anything immediately, and may be called from any thread. The actual
  application happens later, only from the *window thread*, in
  `process_events()` → `set_properties_now()`. Properties that couldn't be
  applied end up in `_rejected_properties`, retrievable via
  `get_rejected_properties()` / cleared via `clear_rejected_properties()`.
  `is_closed()`/property getters read the live `_properties`, which lags
  requests by up to a frame — the header comment on `is_closed()` says this
  explicitly: neither open nor close happens "immediately" after the
  corresponding `GraphicsEngine` call.
- **`set_properties_now()` treats an open/close transition specially.** If
  `properties.has_open()` and it differs from the current open state, *all*
  of `properties` is applied at once via `open_window()`/`close_window()`
  and the method returns early — size/origin/fullscreen/mouse-mode changes
  bundled into the same request are carried along inside
  `_properties` rather than negotiated individually. Only when the window is
  already open (and staying open) does the method negotiate size/origin via
  `do_reshape_request()` and clear only the sub-properties that were
  actually handled, leaving the rest in `properties` for the caller
  (`process_events()`) to log as rejected.
- **Opening failure closes the window, not just leaves it unopened**: if
  `open_window()` returns false, `_rejected_properties` absorbs the whole
  attempted `_properties` set, `_properties.set_open(false)`, and
  `_is_valid = false` — so a failed open looks identical to a subsequently
  closed window from the outside.
- **`is_active()` overrides `GraphicsOutput::is_active()`** to additionally
  require the window be open and not minimized — a valid-but-minimized or
  valid-but-unopened window is inactive even if the base class would
  otherwise say yes.
- **`set_unexposed_draw(false)` trades correctness for efficiency**: with it
  false, the window only redraws after a recent expose/draw event from the
  OS, which avoids wasted rendering while hidden but can leave stale content
  visible in some edge cases; `true` (the `win-unexposed-draw` default)
  always redraws every frame regardless of exposure state.
- **`system_changed_properties()`/`system_changed_size()` are the reverse
  channel** — called *from* platform subclasses (window thread only) when
  the OS itself changes something (user resize, user close), not in
  response to an app request. `system_changed_properties()` throws the
  `window_event` (default `"window-event"`) only if the merged properties
  actually differ from before; `system_changed_size()` is required to run
  *before* the `_properties` size members are updated, since it recalculates
  `DisplayRegion`s against the old vs. new size.
- **`add_input_device()` is the only way `_input_devices` grows**, and it's
  protected — subclasses (or `CallbackGraphicsWindow::create_input_device()`)
  add devices during window setup; there's no public API to add one after
  the fact from application code.
- **Base-class stubs return "unsupported" rather than asserting**:
  `open_window()` returns `false`, `do_reshape_request()` returns `false`,
  `move_pointer()` returns `false`, `verify_window_sizes()` claims all sizes
  are valid, `get_keyboard_map()` returns `nullptr`, `supports_window_procs()`
  returns `false` — a platform subclass overrides whichever of these it
  actually implements; unoverridden ones are safe, inert no-ops.
- **`_rejected_properties`/`_requested_properties`/`_window_event` are
  guarded by a separate `LightReMutex` (`_properties_lock`) from
  `_input_devices` (guarded by `_input_lock`)** — the two subsystems never
  contend with each other.

## API

### Properties

| Signature | Notes |
|---|---|
| `const WindowProperties get_properties() const` | Live, applied properties. |
| `const WindowProperties get_requested_properties() const` | Pending, not-yet-applied requests. |
| `void request_properties(const WindowProperties &requested_properties)` | Thread-safe; merges into the pending request. Actually applied at the next `process_events()` in the window thread. |
| `void clear_rejected_properties()` / `WindowProperties get_rejected_properties() const` | |
| `INLINE bool is_closed() const` | See behavior notes on the async lag. |
| `virtual bool is_active() const` | Overrides `GraphicsOutput::is_active()`; also requires open + not minimized. |
| `INLINE bool is_fullscreen() const` | |
| `void set_window_event(const string&)` / `string get_window_event() const` | Default `"window-event"`, thrown with `this` as the sole parameter. |
| `void set_close_request_event(const string&)` / `string get_close_request_event() const` | Empty (default) = Panda closes the window itself on a close request; nonempty = the event fires instead and the app must call `request_close()`/`close_window()` explicitly. |
| `INLINE void set_unexposed_draw(bool)` / `INLINE bool get_unexposed_draw() const` | Default from `win-unexposed-draw` config var. |
| `INLINE WindowHandle *get_window_handle() const` | See [WindowHandle.md](WindowHandle.md). |

### Input devices

| Signature | Notes |
|---|---|
| `int get_num_input_devices() const` / `InputDevice *get_input_device(int) const` / `string get_input_device_name(int) const` | Thread-safe (`_input_lock`). |
| `bool has_pointer(int) const` / `bool has_keyboard(int) const` | |
| `virtual ButtonMap *get_keyboard_map() const` | Base returns `nullptr`; platform subclasses report raw→virtual button mapping. |
| `void enable_pointer_events(int)` / `void disable_pointer_events(int)` | |
| `virtual MouseData get_pointer(int) const` | Deprecated for devices other than 0; see `InputDeviceManager` (outside this module) for raw multi-device input. |
| `virtual bool move_pointer(int, int, int)` | Base returns `false`; not all platforms/APIs support warping the cursor. |
| `virtual void close_ime()` | Base no-op; platforms with IME support override to force-close any open input-method window. |

### Lifecycle (mostly called by `GraphicsEngine`, not application code)

| Signature | Notes |
|---|---|
| `virtual void request_open()` / `virtual void request_close()` | Async; queue an open/close via `request_properties()`. |
| `virtual void set_close_now()` | Synchronous, window-thread-only; forces immediate close. |
| `virtual void process_events()` | Window-thread-only. Pumps OS messages and applies any pending `request_properties()`. |
| `virtual void set_properties_now(WindowProperties&)` | Window-thread-only; synchronous property application. See behavior notes. |
| `virtual void close_window()` (protected) / `virtual bool open_window()` (protected) | Base `open_window()` always fails (`return false`); real work is in platform subclasses. |
| `virtual void reset_window(bool swapchain)` (protected) | Recreates the framebuffer after a lost device/swapchain reset; base is a no-op logging stub. |
| `virtual bool do_reshape_request(...)` (protected) | Base returns `false`. |
| `virtual void mouse_mode_absolute()` / `virtual void mouse_mode_relative()` (protected) | Base no-ops; platforms with raw/relative mouse capture override. |

### Touch support

| Signature | Notes |
|---|---|
| `virtual bool is_touch_event(GraphicsWindowProcCallbackData *callbackData)` | Base returns `false`. |
| `virtual int get_num_touches()` | Base returns `0`. |
| `virtual TouchInfo get_touch_info(int index)` | Base returns a default-constructed `TouchInfo`. See **TouchInfo** subsection below. |

### Window-proc callback hooks

| Signature | Notes |
|---|---|
| `virtual void add_window_proc(const GraphicsWindowProc *)` / `virtual void remove_window_proc(const GraphicsWindowProc *)` / `virtual void clear_window_procs()` | Base implementations are empty no-ops; only platforms that actually support this (Windows) override them meaningfully. |
| `virtual bool supports_window_procs() const` | Base returns `false`. |

### Misc

| Signature | Notes |
|---|---|
| `virtual int verify_window_sizes(int numsizes, int *dimen)` | Base claims every requested size is valid; real fullscreen-mode validation happens in platform subclasses. |

## GraphicsWindowProc / GraphicsWindowProcCallbackData

`GraphicsWindowProc` (`graphicsWindowProc.h/.cxx`) is a near-empty abstract
interface: a virtual `wnd_proc(GraphicsWindow*, HWND, UINT, WPARAM, LPARAM)`
method, compiled only on Windows (`__WIN32__`/`_WIN32`), that a platform
window's message loop calls for every raw OS window message before/instead
of Panda's default handling. It exists so app code can intercept native
window messages (e.g. custom `WM_*` handling) via
`GraphicsWindow::add_window_proc()`/`remove_window_proc()` — but note the
base `GraphicsWindow` doesn't implement those hooks at all
(`supports_window_procs()` returns `false`); only a platform-specific
`GraphicsWindow` subclass that actually stores and dispatches to registered
`GraphicsWindowProc` objects makes this mechanism do anything. (The
`PythonGraphicsWindowProc` subclass — Python-only glue — is the only
concrete implementation in this module; excluded from these docs.)

`GraphicsWindowProcCallbackData` (a `CallbackData` subclass — see
[../event/README.md](../event/README.md) for the general Panda callback
pattern used elsewhere) is what actually gets delivered through Panda's
generic callback path when a window-proc-style event occurs; on Windows it
additionally carries the raw `hwnd`/`msg`/`wparam`/`lparam`. Its
`is_touch_event()`/`get_num_touches()`/`get_touch_info()` are thin
forwarders onto the owning `GraphicsWindow`'s same-named virtuals (see
Touch support above) — it doesn't hold touch state itself.

## TouchInfo

A small `PUBLISHED` value class (`touchInfo.h/.cxx`) describing one touch
point: `_x`, `_y`, `_id` (to correlate move/down/up events for the same
finger across calls), and `_flags` (a bitmask of `TIF_move`/`TIF_down`/
`TIF_up`). It has no logic beyond getters/setters — plain data. Nothing in
this module's base classes actually populates one; `GraphicsWindow::get_touch_info()`
and `get_num_touches()` are stub `virtual`s returning `0`/a default
`TouchInfo()`, so a platform window subclass with real touch/multitouch
support (Windows 8+ touch APIs) is expected to override them and hand back
real `TouchInfo` values indexed `0..get_num_touches()-1`.

## Usage

```cpp
// Normally you never construct a GraphicsWindow directly:
GraphicsWindow *win = DCAST(GraphicsWindow, engine->make_output(pipe, "win", 0,
    fb_props, win_props, GraphicsPipe::BF_require_window));

win->request_properties(WindowProperties::size(1024, 768));

// Each frame (or in a task), from the window thread:
win->process_events();
if (win->is_closed()) {
  // user closed it, or open failed
}
```

## See also

- [GraphicsOutput.md](GraphicsOutput.md) — base class (framebuffer, `DisplayRegion`s, frame lifecycle).
- [CallbackGraphicsWindow.md](CallbackGraphicsWindow.md), [SubprocessWindow.md](SubprocessWindow.md) — non-native-OS-window subclasses.
- [GraphicsWindowInputDevice.md](GraphicsWindowInputDevice.md) — what populates `_input_devices`.
- [WindowProperties.md](WindowProperties.md), [WindowHandle.md](WindowHandle.md)
- [../framework/WindowFramework.md](../framework/WindowFramework.md) — wraps exactly one `GraphicsWindow`/`GraphicsOutput`.
- [README.md](README.md) — `win-unexposed-draw` and other window-default config variables.
