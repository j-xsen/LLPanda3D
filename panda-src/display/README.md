# panda/src/display

The rendering-target and device-abstraction layer: everything about opening
a window/buffer, the pipeline that drives frame rendering across it, and the
base class that every real graphics API (GL, DX, etc.) subclasses to
implement actual drawing. This is the largest and most central module
documented so far — `GraphicsEngine` and `GraphicsStateGuardian` alone are
each bigger than most entire modules covered previously. A specific 3-D API
(OpenGL, DirectX, TinyDisplay) lives in its own module (`panda/src/glstuff`,
`panda/src/dxgsg9`, etc.) and derives from the base classes documented here
(`GraphicsPipe`, `GraphicsStateGuardian`, `GraphicsWindow`/`GraphicsBuffer`);
those subclasses are **not** covered by this module.

## Class hierarchy

```
ReferenceCount
  └── GraphicsEngine            (the orchestrator: owns pipes, windows, the
                                  render pipeline/threading, frame stats)

TypedReferenceCount
  ├── GraphicsPipe               (one physical display adapter/API binding)
  │     └── (uses) GraphicsDevice   (folded into GraphicsPipe.md)
  ├── GraphicsDevice              (vendor/adapter handle; folded into GraphicsPipe.md)
  ├── DisplayRegion + DrawableRegion
  │     └── StereoDisplayRegion
  └── WindowHandle
        └── NativeWindowHandle    (folded into WindowHandle.md)

GraphicsOutputBase (external base) + DrawableRegion
  └── GraphicsOutput              (one renderable target: a window OR an
                                    offscreen buffer)
        ├── GraphicsBuffer         (offscreen render-to-texture target)
        ├── ParasiteBuffer         (renders into a sub-rectangle of another
        │                           GraphicsOutput's texture memory)
        └── GraphicsWindow         (an actual onscreen OS window)
              ├── CallbackGraphicsWindow  (window driven by app-supplied
              │                            callbacks instead of a real OS window)
              └── SubprocessWindow        (renders in a child process into
                                            shared memory the parent displays;
                                            SubprocessWindowBuffer folded in)

  GraphicsWindow also folds in GraphicsWindowProc / GraphicsWindowProcCallbackData
  (OS-message callback interface) and TouchInfo (touch-point data, coupled to
  GraphicsWindow's own stub virtuals rather than to GraphicsWindowInputDevice).

GraphicsStateGuardianBase (external base)
  └── GraphicsStateGuardian       (GSG: the abstract base every real 3-D API
                                    backend subclasses — render state,
                                    textures, geometry submission)

StateMunger (external base)
  └── StandardMunger               (default GeomVertexData munger the GSG
                                     uses to adapt geometry to hardware limits)

InputDevice (external base, panda/src/device)
  └── GraphicsWindowInputDevice    (one keyboard/mouse device on a window)

DataNode (external base, panda/src/dgraph)
  └── MouseAndKeyboard             (data-graph node that outputs a window's
                                     mouse+keyboard state)

CallbackData (external base, panda/src/putil)
  ├── DisplayRegionCullCallbackData  (folded into DisplayRegion.md)
  ├── DisplayRegionDrawCallbackData  (folded into DisplayRegion.md)
  └── GraphicsWindowProcCallbackData (folded into GraphicsWindow.md)

Standalone value/utility classes (no ref-counted base):
  GraphicsPipeSelection   — registry + factory for available GraphicsPipe types
  GraphicsThreadingModel  — parses/represents a "cull/draw"-style pipeline spec
  WindowProperties        — requested/actual window config (size, title, flags...)
  FrameBufferProperties   — requested/actual framebuffer config (depth/color/stencil bits...)
  DisplayInformation      — queried hardware/display capabilities (DisplaySearchParameters folded in)
  RenderBuffer            — folded into GraphicsStateGuardian.md
  GraphicsWindowProc      — abstract OS-message callback interface, folded into GraphicsWindow.md
```

## Files excluded from this documentation

- `*_ext.h` / `*_ext.cxx` (`graphicsStateGuardian_ext`, `graphicsWindow_ext`,
  `windowProperties_ext`) — Python scripting-interface glue.
- `pythonGraphicsWindowProc.h/.cxx` — a `GraphicsWindowProc` subclass that
  dispatches to a Python callable; Python-only.
- `p3display_composite1.cxx`, `p3display_composite2.cxx`,
  `p3display_ext_composite.cxx` — build-system unity/composite translation
  units (just `#include` the real `.cxx` files for one compile unit).
- `get_x11.h`, `pre_x11_include.h`, `post_x11_include.h` — X11 header
  include-order workarounds, not API.
- `subprocessWindowBuffer.h/.cxx/.I` — internal helper used only by
  `SubprocessWindow`; documented as a subsection of `SubprocessWindow.md`
  rather than standalone.

## Config variables (`config_display.h`)

Two Notify categories: `display` (general) and `gsg` (GSG-specific,
subcategory of `display`).

**Threading / engine:**

| Variable | Default | Meaning |
|---|---|---|
| `threading-model` | `""` | Default pipeline threading spec for new windows (see `GraphicsThreadingModel.md`); experimental, single-threaded (`""`) is the only well-supported value. |
| `allow-nonpipeline-threads` | `false` | Debug-only: allow a threaded model even without pipelining compiled in. |
| `auto-flip` | `false` | If true, `render_frame()` flips all windows before returning (single-threaded mode only). |
| `sync-flip` | `false` | If true, try to flip all windows simultaneously rather than each as late as possible. |
| `yield-timeslice` | `false` | Yield the OS timeslice at end of frame. |
| `subprocess-window-max-wait` | `0.2` | Seconds `SubprocessWindow::begin_flip()` waits for the parent to consume the previous frame. |

**Culling / stats:**

| Variable | Default | Meaning |
|---|---|---|
| `view-frustum-cull` | `true` | Disable for debugging only. |
| `pstats-unused-states` | `false` | Adds per-frame overhead to report unused `TransformState`/`RenderState` counts to PStats. |

**Screenshots:**

| Variable | Default | Meaning |
|---|---|---|
| `screenshot-filename` | `"%~p-%a-%b-%d-%H-%M-%S-%Y-%~f.%~e"` | Filename pattern for `save_screenshot_default()`. |
| `screenshot-extension` | `"jpg"` | Default image type for screenshots. |

**Offscreen-buffer strategy** (all consumed by `GraphicsOutput::make_texture_buffer()`/`make_render_texture()`):

| Variable | Default | Meaning |
|---|---|---|
| `prefer-texture-buffer` | `true` | Prefer a real render-to-texture offscreen buffer when the card supports it. |
| `prefer-parasite-buffer` | `false` | Try a `ParasiteBuffer` before a conventional offscreen buffer. |
| `force-parasite-buffer` | `false` | Strongly prefer `ParasiteBuffer` even at the cost of shrinking to fit the window. |
| `prefer-single-buffer` | `true` | Prefer single-buffered offscreen buffers over double-buffered. |

**GSG capability limits/behavior:**

| Variable | Default | Meaning |
|---|---|---|
| `max-texture-stages` | `-1` | Cap on reported texture stages; `-1`/`0` = report the GSG's real number. |
| `max-color-targets` | `-1` | Cap on reported color render targets. |
| `support-render-texture` | `true` | Allow true render-to-texture; else offscreen renders copy to a texture. |
| `support-rescale-normal` | `true` | Allow uniform-scale normal rescaling instead of renormalizing (GL only). |
| `support-stencil` | `true` | Allow stencil buffer use if the card supports it. |
| `copy-texture-inverted` | `false` | Override whether the GSG's framebuffer-to-texture copy inverts. |
| `color-scale-via-lighting` | `true` | Implement `ColorAttrib`/`ColorScaleAttrib` via a synthetic material/ambient light rather than munging vertex colors. |
| `alpha-scale-via-texture` | `true` | Implement alpha-affecting `ColorScaleAttrib` via an extra texture layer. |
| `allow-incomplete-render` | `true` | Render with paged-out geometry/textures rather than stalling the frame. |
| `old-alpha-blend` | `false` | Restore pre-1.10 (squared) alpha-blend output behavior. |

**Default window properties** (consumed by `WindowProperties::get_default()`):

| Variable | Default | Meaning |
|---|---|---|
| `win-size` | `"800 600"` | Default window size. |
| `win-origin` | `""` | Default window position; `-1`=default, `-2`=centered. |
| `fullscreen` | `false` | |
| `undecorated` | `false` | No title bar/resizable border. |
| `win-fixed-size` | `false` | No resizable border. |
| `cursor-hidden` | `false` | |
| `icon-filename` | `""` | |
| `cursor-filename` | `""` | |
| `z-order` | `WindowProperties::Z_normal` | |
| `window-title` | `"Panda"` | |
| `parent-window-handle` | `0` | Native handle (HWND / NSWindow* / XWindow) to embed the window inside. |
| `win-unexposed-draw` | `true` | Default for `GraphicsWindow::set_unexposed_draw()`. |
| `subprocess-window` | `""` | Path to a `SubprocessWindowBuffer` mmap file (OSX plugin embedding). |
| `window-inverted` | `false` | Render all windows upside-down/backwards (debugging). |

**Stereo:**

| Variable | Default | Meaning |
|---|---|---|
| `red-blue-stereo` | `false` | Anaglyph fallback when true hardware stereo isn't available. |
| `red-blue-stereo-colors` | `"red cyan"` | Channel assignment for the two eyes. |
| `side-by-side-stereo` | `false` | Side-by-side fallback stereo mode. |
| `sbs-left-dimensions` | `"0.0 0.5 0.0 1.0"` | Left-eye region when side-by-side. |
| `sbs-right-dimensions` | `"0.5 1.0 0.0 1.0"` | Right-eye region when side-by-side. |
| `swap-eyes` | `false` | Reverse left/right output. |
| `default-stereo-camera` | `true` | Whether a stereo window/buffer's default `DisplayRegion` is a `StereoDisplayRegion`. |

**Framebuffer request defaults** (consumed by `FrameBufferProperties::get_default()`):

| Variable | Default | Meaning |
|---|---|---|
| `framebuffer-mode` | `""` | Deprecated, no effect. |
| `framebuffer-hardware` | `true` | |
| `framebuffer-software` | `false` | |
| `framebuffer-multisample` | `false` | |
| `framebuffer-depth` | `true` | |
| `framebuffer-alpha` | `true` | |
| `framebuffer-stencil` | `false` | |
| `framebuffer-accum` | `false` | |
| `framebuffer-stereo` | `false` | |
| `framebuffer-srgb` | `false` | |
| `framebuffer-float` | `false` | |
| `depth-bits` / `color-bits` / `alpha-bits` / `stencil-bits` / `accum-bits` / `multisamples` | `0`/`""`/`0`/`0`/`0`/`0` | Minimum-bits requests. |
| `back-buffers` | `1` | |
| `shadow-depth-bits` | `24` | Depth bits specifically for shadow-map buffers. |
| `shadow-cube-map-filter` | `false` | Hardware depth-compare for point-light shadow cube maps; leave false for the shader generator. |

**Misc:**

| Variable | Default | Meaning |
|---|---|---|
| `pixel-zoom` | `1.0` | Default `GraphicsOutput` pixel zoom factor. |
| `background-color` | `"0.41 0.41 0.41 0.0"` | Default clear color for new windows/buffers. |
| `sync-video` | `true` | Sync to video refresh (vsync). |
| `display-zoom` | `0.0` | Overrides detected system DPI scaling when nonzero; read by `GraphicsPipe::get_display_zoom()`. |

`init_libdisplay()` registers every concrete type in this module with the
`TypeRegistry` (needed for `DCAST`/`is_of_type` to work) and, when threading
and pipelining are compiled in, registers a `"pipelining"` capability with
`PandaSystem`.

## Shared concepts

**Engine → Pipe → Output → Region, GSG alongside.** `GraphicsEngine` is the
single top-level object an app owns; it creates `GraphicsOutput`s (windows
or buffers) on a chosen `GraphicsPipe`, and each output is associated with a
`GraphicsStateGuardian` (the actual API-specific renderer) once it opens.
Each output has one or more `DisplayRegion`s carved out of it, and each
region has a `Camera` — the region+camera pair is a "layer" the engine
renders in one pass. See `GraphicsEngine.md` for the full frame lifecycle
(`render_frame()` → per-pipeline cull → draw → flip).

**`DrawableRegion` is the shared base of "things with clear flags."** Both
`GraphicsOutput` (clears the whole target) and `DisplayRegion` (clears just
its sub-rectangle) inherit the same clear-color/clear-depth/clear-stencil/
clear-accum active-flag-plus-value API from `DrawableRegion`. A
`DisplayRegion` only actually clears if its own clear-active flags are set
(see `DisplayRegion.md`) — otherwise the owning `GraphicsOutput`'s clear
(if active) is what paints the background.

**Everything long-lived is reference-counted (`PT()`/`CPT()`).** Like the
rest of Panda3D, ownership of `GraphicsOutput`, `DisplayRegion`,
`GraphicsPipe`, `GraphicsStateGuardian`, and `WindowHandle` instances is
managed via `PointerTo` smart pointers, not manual `delete`.

**Windows/buffers are opened only through `GraphicsEngine::make_output()`**
(the `framework`-module classes wrap this — see
[../framework/README.md](../framework/README.md)); the `GraphicsOutput`
subclass constructors are `protected`/pipe-internal, mirroring the pattern
already seen with `WindowFramework` in the `framework` module.

## See also

- [../framework/README.md](../framework/README.md) — `PandaFramework`/`WindowFramework` are a thin convenience layer built entirely on top of this module's `GraphicsEngine`/`GraphicsPipe`/`GraphicsWindow`/`DisplayRegion`.
- [../event/AsyncTaskManager.md](../event/AsyncTaskManager.md) — `GraphicsEngine::render_frame()` is normally driven from a per-frame task, exactly like the `"igloop"` task set up in `PandaFramework`.
