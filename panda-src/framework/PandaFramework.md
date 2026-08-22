# PandaFramework

**Source:** `panda/src/framework/pandaFramework.h` (+ `.I`, `.cxx`)
**Inherits:** (none) **Inherited by:** (application-specific subclasses, e.g. `pview`'s `ViewerFramework`)

Application-level scaffolding for a simple Panda3D C++ program: owns the
`GraphicsEngine`, the global `EventHandler` and `AsyncTaskManager`, the list
of open `WindowFramework`s, and a set of default viewer keybindings (ESC to
quit, `w` for wireframe, `t` for texture toggle, arrow keys to walk the
scene graph in "highlight" mode, etc.). One instance per application. Not a
required part of using Panda3D — it exists to remove boilerplate from small
programs and samples.

## Behavior notes

- **`open_framework()` is idempotent** — a second call while already open is
  a no-op (checked via `_is_open`). It registers three tasks unconditionally
  (`"event"` at default sort, `"data_loop"` at sort `-50`, and — via the
  lazy `get_graphics_engine()` accessor — `"igloop"` at sort `50` the first
  time the engine is fetched) plus a `"garbageCollectStates"` task at sort
  `46` if the global `garbage_collect_states` config variable is true. Sort
  order matters: data (`-50`) → app tasks → event dispatch → GC (`46`) →
  render (`50`).
- **Recording/playback is mutually exclusive and decided at
  `open_framework()` time** from the `playback_session` / `record_session`
  config variables — `playback_session` is checked first, so if both are
  set, playback wins and `record_session` is ignored.
- **`open_window()` (no-arg overload) fails over across pipe types.** If the
  default pipe can't open a window, it asks `GraphicsPipeSelection` to load
  aux display modules and tries every other registered pipe type in turn,
  promoting the first one that successfully opens a window to
  `_default_pipe`. Only returns `nullptr` if every pipe type fails.
- **`get_graphics_engine()` lazily creates the `"igloop"` render task** the
  first time it's called (in the `.I` file, not obviously from the header
  alone) — so calling `get_graphics_engine()` before `open_window()`
  has a side effect of scheduling per-frame rendering.
- **Static `_loader_options`** (`PandaFramework::_loader_options`) is a
  process-wide `LoaderOptions` shared by every `WindowFramework::load_model()`
  call across every `PandaFramework`/`WindowFramework` instance — there's
  only one, not one per framework instance.
- **`event_esc`'s cascade on close.** Closing a window via the default ESC/q
  handler doesn't just close that one `WindowFramework` — it looks up the
  underlying `GraphicsOutput` and closes *every* `WindowFramework` in
  `_windows` that references the same output (relevant after
  `split_window()`, where multiple `WindowFramework`s share one
  `GraphicsOutput`), then removes the shared mouse node, then sets
  `_exit_flag` if that was the last window. `event_window_event` (fired when
  the OS reports the window destroyed, e.g. user clicked the OS close
  button) does the identical cascade-and-maybe-exit logic independently.
- **`define_key()` replaces same-named hooks.** Calling it again with an
  `event_name` already registered removes the old `_key_definitions` entry
  (used to build the `?`-key help text) before adding the new hook — but it
  only replaces the *description* bookkeeping; `EventHandler::add_hook`
  allows multiple hooks per event name, so if `function`/`data` differ, both
  old and new hook still fire unless the old one is separately removed.
- **`do_frame()` just polls the task manager** — `_task_mgr.poll()` — and
  returns `!_exit_flag`. `main_loop()` is a bare `while (do_frame(...)) {}`.
  All real per-frame work (data graph traversal, event dispatch, rendering,
  recording) happens as tasks registered in `open_framework()`/
  `WindowFramework::create_anim_controls()`, not in `do_frame()` itself.
- **`close_framework()` does not clear `_recorder` state gracefully** beyond
  setting the pointer to `nullptr` — no explicit `end_record`/`end_playback`
  call is made; whatever `RecorderController`'s own destructor does on
  release is what finalizes the session file.
- **`task_igloop`'s defensive nullptr-engine check** is a documented
  workaround (see comment in `.cxx`) for a crash when `PandaFramework` is
  destructed (e.g. process exit) while `render_frame()` is executing,
  because some apps (including `pview`) instantiate `PandaFramework` at
  global scope.
- **`set_highlight`/arrow-key navigation** operate purely on `NodePath`
  parent/child/sibling relationships within whatever tree is currently
  highlighted (defaulting to `get_models()`); moving to a sibling explicitly
  guards `node != self->get_models()` so you can't navigate above the models
  root via arrow keys.
- **`hide_collision_solids`/`show_collision_solids` are static and
  recursive**, walking the full subtree looking for `CollisionNode` or
  `OccluderNode` instances; `show_collision_solids` only shows nodes whose
  nearest hidden ancestor is the node itself (`get_hidden_ancestor() == node`),
  i.e. it won't un-hide a node that's hidden because a *parent* is hidden.

## API

### Lifecycle

| Signature | Notes |
|---|---|
| `PandaFramework()` | Binds to the global `EventHandler` and global `AsyncTaskManager`. All flags default off; `_texture_enabled` defaults **true**. |
| `void open_framework()` | Idempotent. See behavior notes. |
| `void open_framework(int &argc, char **&argv)` | Deprecated; forwards to the no-arg overload. |
| `void close_framework()` | Closes all windows, tears down the engine, removes all event hooks, cleans up the task manager, resets flags. Safe to call even if never opened. |
| `virtual ~PandaFramework()` | Calls `close_framework()` if still open. |

### Accessors

| Signature | Notes |
|---|---|
| `GraphicsPipe *get_default_pipe()` | Lazily creates via `make_default_pipe()` on first call. |
| `INLINE GraphicsEngine *get_graphics_engine()` | Lazily creates the engine **and** schedules the `"igloop"` render task on first call. |
| `INLINE const NodePath &get_data_root() const` | Root of the data graph (mouse/keyboard input). |
| `INLINE EventHandler &get_event_handler()` | The framework's `EventHandler` (the process-global one). |
| `INLINE AsyncTaskManager &get_task_mgr()` | The framework's `AsyncTaskManager` (the process-global one). |
| `NodePath get_mouse(GraphicsOutput *window)` | Returns (creating if needed) the shared `MouseAndKeyboard` node for a given output; wraps it in a `MouseRecorder` if a recorder is active. |
| `void remove_mouse(const GraphicsOutput *window)` | Removes and forgets the mouse node created above. |
| `NodePath &get_models()` | Lazily-created root node meant to parent loaded models. |
| `INLINE bool has_highlight() const` / `INLINE const NodePath &get_highlight() const` | Current highlighted node, if any. |
| `INLINE RecorderController *get_recorder() const` / `INLINE void set_recorder(RecorderController *)` | Must be set before opening windows to take effect on their input. |

### Keys and windows

| Signature | Notes |
|---|---|
| `void define_key(const string &event_name, const string &description, EventHandler::EventCallbackFunction *function, void *data)` | Registers a keybinding + help-text entry. See behavior notes on replacement semantics. |
| `INLINE void set_window_title(const string &title)` | Applies to windows opened after this call. |
| `virtual void get_default_window_props(WindowProperties &props)` | Merges `WindowProperties::get_default()` with the configured title. |
| `WindowFramework *open_window()` | Opens on the default pipe, failing over across pipe types. See behavior notes. |
| `WindowFramework *open_window(GraphicsPipe *pipe, GraphicsStateGuardian *gsg = nullptr)` | Uses default window properties; respects `window_type == "offscreen"`. |
| `WindowFramework *open_window(const WindowProperties &props, int flags, GraphicsPipe *pipe = nullptr, GraphicsStateGuardian *gsg = nullptr)` | Full control. Propagates the framework's current wireframe/texture/two-sided/lighting/perpixel/background settings onto the new `WindowFramework`. |
| `INLINE int get_num_windows() const` / `INLINE WindowFramework *get_window(int n) const` | |
| `int find_window(const GraphicsOutput *win) const` / `int find_window(const WindowFramework *wf) const` | Returns `-1` if not found. |
| `void close_window(int n)` / `INLINE void close_window(WindowFramework *wf)` | |
| `void close_all_windows()` | |
| `bool all_windows_closed() const` | True if every open `WindowFramework`'s `GraphicsOutput::is_valid()` is false. |

### Global rendering toggles (apply to all current windows + become the default for new ones)

| Signature |
|---|
| `void set_wireframe(bool)` / `INLINE bool get_wireframe() const` |
| `void set_texture(bool)` / `INLINE bool get_texture() const` |
| `void set_two_sided(bool)` / `INLINE bool get_two_sided() const` |
| `void set_lighting(bool)` / `INLINE bool get_lighting() const` |
| `void set_perpixel(bool)` / `INLINE bool get_perpixel() const` |
| `void set_background_type(WindowFramework::BackgroundType)` / `INLINE WindowFramework::BackgroundType get_background_type() const` |

### Scene graph utilities

| Signature | Notes |
|---|---|
| `static int hide_collision_solids(NodePath node)` | Recursive; returns count of nodes newly hidden. |
| `static int show_collision_solids(NodePath node)` | Recursive; only un-hides directly-hidden nodes. |
| `void set_highlight(const NodePath &node)` | Shows bounds + filled-wireframe render mode on the node. |
| `void clear_highlight()` | |

### Frame rate / loop

| Signature | Notes |
|---|---|
| `void report_frame_rate(std::ostream &out) const` | Prints frames-since-reset and average fps. |
| `void reset_frame_rate()` | |
| `void enable_default_keys()` | Idempotent; calls `do_enable_default_keys()` once. |
| `virtual bool do_frame(Thread *current_thread)` | Polls the task manager; returns `!_exit_flag`. |
| `void main_loop()` | `while (do_frame(...)) {}`. |
| `INLINE void set_exit_flag()` / `INLINE void clear_exit_flag()` | |

### Extension hooks (protected virtuals)

| Signature | Notes |
|---|---|
| `virtual PT(WindowFramework) make_window_framework()` | Default: `new WindowFramework(this)`. Override to inject a custom `WindowFramework` subclass. |
| `virtual void make_default_pipe()` | Default: uses `GraphicsPipeSelection::get_global_ptr()->make_default_pipe()`. |
| `virtual void do_enable_default_keys()` | Default: registers the full standard keymap (see below). |

## Default keymap (`do_enable_default_keys()`)

| Key | Action |
|---|---|
| `escape`, `q` | Close current window (cascades to sibling split-windows on the same output; exits app if last window) |
| `f` | Report + reset frame rate |
| `w` | Toggle wireframe (cycles: off → wireframe → filled-wireframe → off) |
| `t` | Toggle texturing |
| `b` | Toggle two-sided (backface) rendering |
| `i` | Toggle one-sided-reverse (invert culled side) |
| `l` | Toggle lighting |
| `p` | Toggle per-pixel (shader-auto) lighting |
| `c` | Center trackball on highlighted node (or whole model tree) |
| `a` | Cycle to next animation control |
| `shift-c` | Toggle collision-solid/occluder visibility |
| `shift-b` | Print bounding volume of highlighted node/scene to `nout` |
| `shift-l` | `NodePath::ls()` the highlighted node/scene |
| `shift-a` | Run `SceneGraphAnalyzer` on the highlighted node/scene, print to `nout` |
| `h` | Toggle highlight mode |
| `arrow_up/down/left/right` | In highlight mode: move to parent/first child/left sibling/right sibling |
| `shift-s` | Connect to PStats (if `DO_PSTATS` built in) |
| `f9` | Save + display an onscreen screenshot filename for 3 seconds |
| `,` | Cycle background type (default → black → gray → white → default) |
| `?`, `shift-/` | Toggle onscreen key-help listing |

Every one of these handlers is wired through `define_key`, receives the
originating `WindowFramework*` as `event->get_parameter(0)` (for keys) except
the window-event handler, which receives the `GraphicsOutput*`.

## Usage

```cpp
#include "pandaFramework.h"
#include "pandaSystem.h"

int main(int argc, char *argv[]) {
  PandaFramework framework;
  framework.open_framework(argc, argv);
  framework.set_window_title("My App");

  WindowFramework *window = framework.open_window();
  if (window != nullptr) {
    window->enable_keyboard();
    window->setup_trackball();
    framework.enable_default_keys();

    NodePath model = window->load_model(framework.get_models(), "models/panda");
    window->loop_animations(0);
  }

  framework.main_loop();
  framework.close_framework();
  return 0;
}
```

## See also

- [WindowFramework.md](WindowFramework.md) — every window `PandaFramework` opens.
- [README.md](README.md) — config variables, shared 2-d/mouse concepts.
- [../event/AsyncTaskManager.md](../event/AsyncTaskManager.md), [../event/EventHandler.md](../event/EventHandler.md) — the two subsystems `PandaFramework` wraps.
