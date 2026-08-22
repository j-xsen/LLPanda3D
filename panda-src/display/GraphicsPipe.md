# GraphicsPipe

**Source:** `panda/src/display/graphicsPipe.h` (+ `.I`, `.cxx`)
**Inherits:** TypedReferenceCount **Inherited by:** concrete API-specific pipes (`wglGraphicsPipe`, `glxGraphicsPipe`, etc. — live in other modules, not documented here)

An object that creates `GraphicsOutput`s (windows/buffers) sharing a
particular 3-D API and windowing-system binding. Normally an application
has exactly one `GraphicsPipe`; multiple are possible if several different
APIs/adapters are available on the same machine. `GraphicsPipe` itself is
abstract — this base class implements the shared bookkeeping (display
size, hardware-info collection, device handle) and every actual
window/buffer-creation call (`make_output`) is a virtual that concrete
subclasses in the platform-specific display modules override; this base's
own `make_output()` just logs an error and returns `nullptr`.
`GraphicsPipe` keeps ownership of the windows/buffers it creates and is
used by `GraphicsEngine` to create and destroy them — see
`GraphicsEngine::make_output()` in [GraphicsEngine.md](GraphicsEngine.md).
The constructor is `protected` and `get_interface_name()` is pure virtual,
so `GraphicsPipe` is never instantiated directly — only through a concrete
subclass.

## Behavior notes

- **The constructor eagerly collects hardware info** into a heap-allocated
  `DisplayInformation` (`_display_information`, owned and `delete`d by the
  destructor) — CPU vendor string/brand string/features via `cpuid` on
  x86/x64, and OS-specific memory stats (macOS via `sysctlbyname`, Linux via
  `sysinfo()`/stashing an `update_memory_info` refresh callback into
  `_get_memory_information_function`, FreeBSD via `sysctlbyname`, Windows
  via `GlobalMemoryStatusEx`). This happens for *every* `GraphicsPipe`
  instance, not lazily — see [DisplayInformation.md](DisplayInformation.md)
  for what's captured and how it's later refreshed.
- **`_is_valid` starts `true`.** A derived class is responsible for setting
  it `false` if it determines the pipe can't actually be used (e.g. no
  compatible hardware/driver found during its own constructor). Likewise
  `_supported_types` starts at `0` — a derived class must OR in the
  `OutputTypes` bits (`OT_window`, `OT_fullscreen_window`, `OT_buffer`,
  `OT_texture_buffer`) it can actually produce; `supports_type()` is a
  simple bitmask-subset test against whatever the subclass declared.
- **`BufferCreationFlags` is a much larger, orthogonal flag set** from
  `OutputTypes` — it's the request-shaping vocabulary passed into
  `make_output()` (`BF_require_window`, `BF_refuse_parasite`,
  `BF_size_power_2`, `BF_can_bind_layered`, etc.), consumed both by this
  base's `make_output()` signature and by every subclass's actual
  implementation; `GraphicsOutput::make_texture_buffer()`/`make_render_texture()`
  build these flag combinations from the `prefer-texture-buffer`/
  `prefer-parasite-buffer`/`force-parasite-buffer`/`prefer-single-buffer`
  config variables (module [README.md](README.md)).
- **`get_display_zoom()` prefers the config override, falls back to a
  process-wide detected value.** If the `display-zoom` config variable
  (README) is set to a nonzero value, that wins; otherwise it returns a
  `static PN_stdfloat detected_display_zoom` — a genuinely *process-global*
  variable (not per-pipe, despite living in this `.cxx` file), initialized
  to `1.0` and updatable only by a derived class calling the `protected`
  `set_detected_display_zoom()`. Every `GraphicsPipe` instance in the
  process therefore reports the same detected zoom once any one of them
  sets it.
- **`make_callback_gsg()` returning non-null is what makes a
  `CallbackGraphicsWindow` possible** — see
  [CallbackGraphicsWindow.md](CallbackGraphicsWindow.md). The base
  implementation returns `nullptr` (not supported); a pipe that supports it
  constructs a GSG bound to an externally-managed graphics context rather
  than one tied to a real window.
- **`close_gsg()` runs on the draw thread**, symmetric with GSG creation —
  it's the pipe's hook to release any pipe-level resources tied to a GSG
  just before that GSG is destructed; the base implementation just forwards
  to `gsg->close_gsg()`.
- **`lookup_cpu_data()` is a no-op in the base class** — deliberately
  separate from the constructor's basic cpuid/memory collection, meant to
  be overridden to do a slower, more detailed CPU lookup ("this may take a
  second or two", per the doc comment) only when explicitly requested via
  `GraphicsPipe::get_display_information()` + a manual call.

## API

| Signature | Notes |
|---|---|
| `enum OutputTypes { OT_window, OT_fullscreen_window, OT_buffer, OT_texture_buffer }` | Bitmask; queried via `get_supported_types()`/`supports_type()`. |
| `enum BufferCreationFlags { BF_refuse_parasite, BF_require_parasite, BF_refuse_window, BF_require_window, BF_require_callback_window, BF_can_bind_color, BF_can_bind_every, BF_resizeable, BF_size_track_host, BF_rtt_cumulative, BF_fb_props_optional, BF_size_square, BF_size_power_2, BF_can_bind_layered }` | Request-shaping flags for `make_output()`. |
| `INLINE bool is_valid() const` | |
| `INLINE int get_supported_types() const` / `INLINE bool supports_type(int flags) const` | |
| `INLINE int get_display_width() const` / `INLINE int get_display_height() const` | May be `0` if unknown; not a hard cap on window size. |
| `PN_stdfloat get_display_zoom() const` | See behavior notes — config override or process-global detected value. |
| `DisplayInformation *get_display_information()` | Returns the pipe-owned `DisplayInformation`. |
| `virtual void lookup_cpu_data()` | No-op by default; override for a slow detailed CPU query. |
| `virtual std::string get_interface_name() const = 0` | Pure virtual — what makes `GraphicsPipe` abstract; identifies the API (e.g. `"OpenGL"`). |
| `virtual PreferredWindowThread get_preferred_window_thread() const` | `PWT_app` or `PWT_draw`; base default is `PWT_draw`. |
| `INLINE GraphicsDevice *get_device() const` | See the `GraphicsDevice` subsection below. |
| `virtual PT(GraphicsDevice) make_device(void *scrn = nullptr)` | Base logs an error and returns `nullptr`; only meaningfully implemented by DirectX-family pipes. |
| `virtual PT(GraphicsStateGuardian) make_callback_gsg(GraphicsEngine *engine)` | See behavior notes. |
| `virtual void close_gsg(GraphicsStateGuardian *gsg)` (protected) | |
| `virtual PT(GraphicsOutput) make_output(...)` (protected) | The actual window/buffer factory; base implementation always fails. |

## Subsection: GraphicsDevice

**Source:** `panda/src/display/graphicsDevice.h/.I/.cxx`
**Inherits:** TypedReferenceCount

A lighter-weight companion object referenced by `GraphicsPipe::_device`
(`PT(GraphicsDevice)`, one-directional — `GraphicsDevice::_pipe` is a raw
back-pointer, not ref-counted). Per the header comment: "set to NULL for
OpenGL. But DirectX uses it to take control of multiple windows under
[a] single device or multiple devices (i.e. more than one adapter in the
machine)." In other words this base's own `make_device()` is unimplemented
and returns `nullptr` — only DirectX-family pipe subclasses (outside this
module) actually construct and use one, to coordinate several windows that
must share one underlying D3D device object or to represent multiple
physical adapters. Public surface is minimal: `get_pipe()` to recover the
owning pipe; construction takes the owning `GraphicsPipe*` and is otherwise
just a `TypedReferenceCount` with no independent behavior defined in this
module.

## Usage

```cpp
// Typical: never touched directly. GraphicsPipeSelection creates the pipe;
// GraphicsEngine::make_output() uses it to create windows/buffers.
PT(GraphicsPipe) pipe = GraphicsPipeSelection::get_global_ptr()->make_default_pipe();
if (pipe != nullptr && pipe->is_valid()) {
  nout << "Using pipe: " << pipe->get_interface_name() << "\n";
}
```

## See also

- [GraphicsPipeSelection.md](GraphicsPipeSelection.md) — discovers and instantiates `GraphicsPipe` subclasses.
- [GraphicsEngine.md](GraphicsEngine.md) — the actual caller of `make_output()`.
- [DisplayInformation.md](DisplayInformation.md) — what `_display_information` contains.
- [CallbackGraphicsWindow.md](CallbackGraphicsWindow.md) — enabled by `make_callback_gsg()`.
- [../framework/PandaFramework.md](../framework/PandaFramework.md) — `make_default_pipe()` wraps `GraphicsPipeSelection`.

## Uncertain / worth double-checking

`GraphicsPipe`'s constructors/destructor are declared `PUBLISHED`/`public`
in the header snippet actually read (`protected: GraphicsPipe(); ... PUBLISHED: virtual ~GraphicsPipe();`),
i.e. the default constructor is `protected` (so only a derived class can
construct one) while the destructor is public — I described this correctly
above but flagging it since it's an easy detail to get backwards when
skimming.
