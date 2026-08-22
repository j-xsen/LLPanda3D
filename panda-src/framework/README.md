# panda/src/framework

High-level scaffolding for small C++ Panda3D programs (`pview`, sample apps,
test programs). Two classes: **PandaFramework** (app-level: engine, event
handler, task manager, window list, default keybindings) and
**WindowFramework** (one open window or one display region within a split
window: scene graph roots, camera, trackball, mouse/keyboard wiring, model
loading, anim-control UI). Neither class does anything a hand-rolled app
couldn't do directly with `GraphicsEngine`/`NodePath`/`EventHandler` — this
module exists to save boilerplate, and most Panda3D C++ applications embed
this pattern almost verbatim.

## Class hierarchy

```
PandaFramework          (standalone; owns 0+ WindowFramework via PT())
TypedWritableReferenceCount
  └── WindowFramework    (one per open window, or one per split region
                          sharing a GraphicsOutput)
```

`WindowFramework`'s constructors are `protected` — instances are only ever
created via `PandaFramework::open_window()` (which calls the virtual
`make_window_framework()` hook) or `WindowFramework::split_window()`. There
is no public "make a WindowFramework yourself" path.

## Files

- `pandaFramework.h/.I/.cxx` — `PandaFramework`.
- `windowFramework.h/.I/.cxx` — `WindowFramework`.
- `config_framework.h/.cxx` — config variables + the `framework` Notify
  category, folded into this README (below) rather than given a doc file.
- `p3framework_composite1.cxx` — build-system composite unit (just
  `#include`s the two `.cxx` files above for a single compilation unit);
  not documented separately.
- `rock_floor.rgb_src.c`, `shuttle_controls.bam_src.c` — generated C arrays
  of baked asset data (a texture image and a bam-encoded model), `#include`d
  directly into `windowFramework.cxx`. Not source to document; see their use
  in `WindowFramework::load_default_model()` and
  `WindowFramework::get_shuttle_controls_font()` below.

## Config variables (`config_framework.h`)

| Variable | Default | Meaning |
|---|---|---|
| `aspect_ratio` (Double) | `0.0` | Forces window aspect ratio; `0.0` means infer from the window's actual pixel size. Read by `WindowFramework::get_aspect_2d()`, `adjust_dimensions()`, and `make_camera()`. |
| `show_frame_rate_meter` (Bool) | `false` | If true, `WindowFramework::open_window()` creates a `FrameRateMeter` on the window. |
| `show_scene_graph_analyzer_meter` (Bool) | `false` | If true, `WindowFramework::open_window()` creates a `SceneGraphAnalyzerMeter` on the window. |
| `print_pipe_types` (Bool) | `true` | If true, `PandaFramework::make_default_pipe()` prints the available `GraphicsPipe` types before selecting one. |
| `window_type` (String) | `"onscreen"` | If `"offscreen"`, `PandaFramework::open_window(pipe, gsg)` passes `GraphicsPipe::BF_refuse_window` instead of `BF_require_window`. |
| `record_session` (String) | `""` | Path to write a `RecorderController` session-recording file; if non-empty, `open_framework()` creates and starts a recorder in record mode. |
| `playback_session` (String) | `""` | Path to a previously-recorded session file; if non-empty (and checked before `record_session`), `open_framework()` creates and starts a recorder in playback mode. |

The `framework` Notify category (`framework_cat` in `.cxx` files) is used
for informational/warning/error logging throughout both classes.

`config_framework.cxx`'s `ConfigureFn` calls `WindowFramework::init_type()`
at static-init time.

## Shared concepts

**Scene graph roots are lazily created.** `WindowFramework::get_render()`,
`get_render_2d()`, `get_aspect_2d()`, `get_pixel_2d()`, `get_camera_group()`,
and `get_mouse()` all follow the same pattern: check an `NodePath` member for
`is_empty()`, and if empty, construct it (and cache it in the member) on
first call. Callers never need to check whether these exist first — just
call the getter.

**One `PandaFramework` per app; N `WindowFramework`s.** A single
`PandaFramework` instance owns the `GraphicsEngine`, the global
`EventHandler`, the global `AsyncTaskManager`, and the `_windows` list. Each
`WindowFramework` wraps one `GraphicsOutput` (or shares one with sibling
`WindowFramework`s after `split_window()`) and owns that window's own scene
graph roots, camera(s), trackball, and UI state. `WindowFramework` keeps a
raw back-pointer to its owning `PandaFramework` (`get_panda_framework()`) and
is `friend`ed by it.

**Typical minimal app** (see [Usage](#usage) in `PandaFramework.md`):
construct a `PandaFramework`, `open_framework()`, `open_window()`,
optionally `enable_default_keys()`, then call `main_loop()` (or drive
`do_frame()` yourself for a custom loop).

**The 2-d node hierarchy.** `render_2d` is the root of everything not
affected by the 3-d camera (depth test/write off, two-sided, materials off),
viewed through an orthographic camera on `DisplayRegion` sort 10 (drawn
after the 3-d region). `aspect_2d` hangs under `render_2d`, is a `PGTop`
(wired to the window's `MouseWatcher` so PGUI widgets work), and is scaled
by `1/aspect_ratio` so a unit square drawn under it isn't stretched by a
non-square window. `pixel_2d` is a separate `PGTop` also under `render_2d`,
offset to the upper-left corner and scaled so children can be positioned in
window pixel units (down-and-right-positive from `(0,0)` at the top-left
corner, i.e. `(x_size, -y_size)` at the bottom-right).

**Mouse/keyboard data-graph wiring.** `PandaFramework::get_mouse(window)`
creates one `MouseAndKeyboard` node per `GraphicsOutput` under
`PandaFramework::get_data_root()`, shared across any `WindowFramework`s that
reference the same window (e.g. after `split_window()`). Each
`WindowFramework::get_mouse()` then attaches its own `MouseWatcher` child
under that shared node, so multiple display regions of one window can each
have independently-clipped mouse tracking when side-by-side stereo is
active. `enable_keyboard()` attaches a `ButtonThrower` under the
`MouseWatcher`, tagged with an `EventParameter(this)` so key event handlers
can recover which `WindowFramework` the key came from.

**Excluded from individual class docs:** none — both real classes
(`PandaFramework`, `WindowFramework`) get their own file below.

## See also

- [PandaFramework.md](PandaFramework.md)
- [WindowFramework.md](WindowFramework.md)
- [../event/README.md](../event/README.md) — `EventHandler`, `AsyncTaskManager`/`GenericAsyncTask` used throughout this module's task/event wiring.
- [../pgui/README.md](../pgui/README.md) — `PGItem`/`PGButton`/`PGSliderBar` used by `WindowFramework`'s anim-control UI.
