# CallbackGraphicsWindow

**Source:** `panda/src/display/callbackGraphicsWindow.h` (+ `.I`, `.cxx`)
**Inherits:** [GraphicsWindow](GraphicsWindow.md) **Inherited by:** (none)

A "window" that doesn't own or represent a real OS window at all — instead,
every lifecycle operation (open, reshape, poll events, begin/end a frame,
flip) is delegated to application-supplied `CallbackObject`s. This is how
Panda renders into a window/context that some other library or engine
already created and owns (e.g. embedding Panda's renderer inside another
toolkit's already-open OpenGL context). Like `GraphicsWindow`, constructed
only via `GraphicsEngine::make_output()`.

## Behavior notes

- **Three independent, optional callback slots**, each defaulting to
  `nullptr`: `events_callback` (polled for OS/system events, from
  `process_events()`), `properties_callback` (property-change requests,
  from `set_properties_now()`), `render_callback` (drawing — shared across
  four different moments, see below). **Any slot left unset falls back to
  the normal `GraphicsWindow` behavior** for that operation
  (`GraphicsWindow::process_events()`, `::set_properties_now()`,
  `::begin_frame()`/`::end_frame()`/`::begin_flip()`/`::end_flip()`) — so a
  `CallbackGraphicsWindow` with no callbacks configured degrades to
  (mostly non-functional, since `open_window()` still assumes external setup)
  ordinary `GraphicsWindow` semantics rather than crashing.
- **One `render_callback` slot serves four distinct call sites**, all
  delivering a `RenderCallbackData` whose `get_callback_type()` disambiguates
  which of `RCT_begin_frame`/`RCT_end_frame`/`RCT_begin_flip`/`RCT_end_flip`
  is happening — the callback must switch on this to know what's being
  asked of it. `RCT_begin_frame` additionally must call
  `data->set_render_flag(bool)` (default `true`) to tell Panda whether to
  actually proceed with rendering this frame or skip it.
- **Every `*CallbackData::upcall()` re-invokes the corresponding
  `GraphicsWindow::` base-class method directly** (not `CallbackGraphicsWindow::`,
  to avoid re-entering the callback) — e.g.
  `EventsCallbackData::upcall()` calls `_window->GraphicsWindow::process_events()`.
  This lets a callback implementation do its own thing and *also* fall back
  to Panda's default handling for anything it doesn't want to special-case,
  simply by calling `data->upcall()`.
- **`begin_frame()` always drives the GSG itself**, callback or not: after
  the render callback (if any) returns a true render flag, it unconditionally
  calls `_gsg->reset_if_new()`, `_gsg->set_current_properties(...)`, and
  `_gsg->begin_frame(...)` — the callback controls *whether* the frame
  proceeds and can do extra host-API setup, but doesn't replace Panda's own
  GSG-level frame bookkeeping.
- **`end_frame()` resets GSG state before invoking the callback** — sets an
  empty `RenderState`/the internal transform and calls
  `_gsg->clear_before_callback()` — specifically so that if the callback (or
  the host application) wants to do its own rendering afterward using the
  same GSG/context, it starts from a clean slate rather than whatever state
  Panda's own rendering left behind.
- **`open_window()` assumes the callback (or host app) has already opened
  the real underlying context** — it does no window creation at all, just
  fills in placeholder `FrameBufferProperties` (RGB color, minimum 16 color
  bits if unset) and returns `true` unconditionally.
- **`do_reshape_request()` also assumes success** — it doesn't resize
  anything itself, just repackages the requested origin/size into a
  `WindowProperties` and calls `system_changed_properties()` (the same
  "OS told us it changed" channel `GraphicsWindow` platform subclasses use)
  and returns `true`. The actual resize is expected to have already happened
  in the host application/toolkit.
- **`create_input_device()` is the only public way to add an input device**
  to a `CallbackGraphicsWindow** — it wraps the protected
  `GraphicsWindow::add_input_device()` (see [GraphicsWindow.md](GraphicsWindow.md))
  around a fresh `GraphicsWindowInputDevice::pointer_and_keyboard()`. The
  callback code is expected to feed events into the returned device via
  `GraphicsWindowInputDevice`'s `button_down()`/`button_up()`/`set_pointer_in_window()`/etc.

## API

### Callback registration

| Signature | Notes |
|---|---|
| `INLINE void set_events_callback(CallbackObject *)` / `clear_events_callback()` / `get_events_callback() const` | Receives `EventsCallbackData`. |
| `INLINE void set_properties_callback(CallbackObject *)` / `clear_properties_callback()` / `get_properties_callback() const` | Receives `PropertiesCallbackData`. |
| `INLINE void set_render_callback(CallbackObject *)` / `clear_render_callback()` / `get_render_callback() const` | Receives `RenderCallbackData`; see behavior notes on the four callback types. |
| `int create_input_device(const string &name)` | Adds a combined pointer+keyboard `GraphicsWindowInputDevice`; returns its index. |

### Callback data classes (nested; `WindowCallbackData` is the common base)

| Class | Delivered to | Key members |
|---|---|---|
| `WindowCallbackData` | (base) | `get_window()` → the `CallbackGraphicsWindow*`. |
| `EventsCallbackData` | `events_callback` | `upcall()` → `GraphicsWindow::process_events()`. |
| `PropertiesCallbackData` | `properties_callback` | `get_properties()` → mutable `WindowProperties&` to consume; `upcall()` → `GraphicsWindow::set_properties_now(...)`. |
| `RenderCallbackData` | `render_callback` | `get_callback_type()` → `RenderCallbackType`; `get_frame_mode()` → `GraphicsOutput::FrameMode` (valid for begin/end_frame); `set_render_flag(bool)`/`get_render_flag()` (begin_frame only, default `true`); `upcall()` dispatches to the matching `GraphicsWindow::` method by callback type. |

`enum RenderCallbackType { RCT_begin_frame, RCT_end_frame, RCT_begin_flip, RCT_end_flip }`

## Overridden lifecycle methods

| Signature | Delegates to (if callback set) | Falls back to |
|---|---|---|
| `virtual bool begin_frame(FrameMode, Thread*)` | `render_callback` (`RCT_begin_frame`) | `GraphicsWindow::begin_frame()` |
| `virtual void end_frame(FrameMode, Thread*)` | `render_callback` (`RCT_end_frame`) | `GraphicsWindow::end_frame()` |
| `virtual void begin_flip()` | `render_callback` (`RCT_begin_flip`) | `GraphicsWindow::begin_flip()` |
| `virtual void end_flip()` | `render_callback` (`RCT_end_flip`) | `GraphicsWindow::end_flip()` |
| `virtual void process_events()` | `events_callback` | `GraphicsWindow::process_events()` |
| `virtual void set_properties_now(WindowProperties&)` | `properties_callback` | `GraphicsWindow::set_properties_now()` |
| `virtual bool open_window()` (protected) | — always assumes success, see behavior notes | |
| `virtual bool do_reshape_request(...)` (protected) | — always assumes success, see behavior notes | |

## Usage

```cpp
class MyRenderCallback : public CallbackObject {
public:
  virtual void do_callback(CallbackData *cbdata) {
    auto *data = DCAST(CallbackGraphicsWindow::RenderCallbackData, cbdata);
    switch (data->get_callback_type()) {
    case CallbackGraphicsWindow::RCT_begin_frame:
      // ...activate the host app's GL context here...
      break;
    default:
      break;
    }
    data->upcall();  // let Panda do its normal thing too
  }
};

GraphicsOutput *out = engine->make_output(pipe, "embedded", 0, fb_props,
    win_props, GraphicsPipe::BF_refuse_window | GraphicsPipe::BF_require_callback_window,
    gsg);
CallbackGraphicsWindow *cbwin = DCAST(CallbackGraphicsWindow, out);
cbwin->set_render_callback(new MyRenderCallback);
```

## See also

- [GraphicsWindow.md](GraphicsWindow.md) — base class; every fallback path lands here.
- [GraphicsEngine.md](GraphicsEngine.md) — `make_output()`, `BF_require_callback_window` flag.
- [SubprocessWindow.md](SubprocessWindow.md) — a different non-native-window `GraphicsWindow` subclass, solving a different embedding problem (cross-process rather than cross-library).
