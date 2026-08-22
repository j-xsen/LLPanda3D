# GraphicsThreadingModel

**Source:** `panda/src/display/graphicsThreadingModel.h` (+ `.I`, `.cxx`)
**Inherits:** (none — standalone value class) **Inherited by:** (none)

Parses and represents a compact string spec for how a window's per-frame
cull and draw work is distributed across threads (e.g. `"cull/draw"`,
`"-cull"`, `""`). Passed to `GraphicsEngine::open_window()`/`set_threading_model()`
(see [GraphicsEngine.md](GraphicsEngine.md)) to control the pipeline for a
particular window; the process-wide default comes from the `threading-model`
config variable (module [README.md](README.md), which also carries the
explicit warning that non-default threading models are experimental and
incomplete in this Panda3D version).

## Behavior notes

- **The constructor's mini-language, from the source comment:** the string
  is `"<cull-name>/<draw-name>"`. The two names are arbitrary labels used
  only to tell threads apart — they don't have to correspond to real OS
  thread objects until a window is actually opened with this model. `""`
  means "the same as the previous stage." So: `""` or `"/"` → cull and draw
  both run in the main/app thread (fully single-threaded — the overwhelmingly
  common case). `"draw"` or `"draw/draw"` → cull and draw share one thread
  named `"draw"`, separate from app. `"/draw"` → cull stays in the app
  thread, draw moves to a separate `"draw"` thread. `"cull/draw"` → three
  distinct thread roles (app, cull, draw). A leading `-` (e.g. `"-cull"`)
  switches to a fused mode: cull and draw run in the *same* thread
  simultaneously with **no** binning/state-sorting — faster and simpler,
  but scene-graph draw order is used as-is, so `CullBinAttrib` sort order
  and alpha sorting are lost. This corresponds to `get_cull_sorting()`
  returning `false`.
- **`update_stages()` derives numeric pipeline stage indices from the
  names**, and is re-run automatically whenever `set_cull_name()`,
  `set_draw_name()`, or `set_cull_sorting()` is called (not just at
  construction). `_cull_stage` is `0` if `_cull_name` is empty (app thread),
  else `1`. If cull-sorting is disabled or `_draw_name` is empty,
  `_draw_name` is forced equal to `_cull_name` (fused mode always collapses
  to one thread); `_draw_stage` equals `_cull_stage` if the names match,
  otherwise `_cull_stage + 1`. These stage numbers are what
  `DisplayRegion::get_screenshot()` compares against the calling thread's
  actual pipeline stage to decide whether it needs to hop to the draw
  thread (see [DisplayRegion.md](DisplayRegion.md)).
- **Changing the model on an existing `GraphicsThreadingModel` object
  doesn't retroactively affect windows already opened with a *copy* of the
  old value** — the setters' doc comments explicitly note "this won't
  change any windows that were already created with this model; this only
  has an effect on newly-opened windows," since `GraphicsEngine` copies the
  model in at window-open time rather than holding a live reference.
- **`is_default()` is stricter than `is_single_threaded()`** —
  single-threaded just means both names are empty; `is_default()`
  additionally requires `_cull_sorting` to be `true` (i.e. not the `-`
  fused-no-sorting variant), matching the literal "default, cull-then-draw
  single-threaded model" phrasing in the doc comment.

## API

| Signature | Notes |
|---|---|
| `GraphicsThreadingModel(const string &model = "")` | Parses the mini-language described above. |
| `std::string get_model() const` | Reconstructs the canonical string form (round-trips through the constructor). |
| `INLINE const string &get_cull_name() const` / `INLINE void set_cull_name(const string&)` | |
| `INLINE int get_cull_stage() const` | `0` = app thread, `1` = separate cull thread. |
| `INLINE const string &get_draw_name() const` / `INLINE void set_draw_name(const string&)` | |
| `INLINE int get_draw_stage() const` | Equal to cull stage if same thread, else `cull_stage + 1`. |
| `INLINE bool get_cull_sorting() const` / `INLINE void set_cull_sorting(bool)` | `false` = fused no-sort mode (the leading-`-` form). |
| `INLINE bool is_single_threaded() const` | Both names empty. |
| `INLINE bool is_default() const` | Single-threaded **and** cull-sorting enabled. |
| `INLINE void output(std::ostream&) const` | Prints `get_model()`. |

## Usage

```cpp
// Explicit single-threaded (the safe, well-supported default):
GraphicsThreadingModel single_threaded;   // "" -> app thread for both

// EXPERIMENTAL: separate cull and draw threads.
GraphicsThreadingModel cull_draw("cull/draw");
engine->open_window(props, flags, pipe, gsg, cull_draw);
```

## See also

- [GraphicsEngine.md](GraphicsEngine.md) — consumes this to set up per-window pipeline stages.
- [DisplayRegion.md](DisplayRegion.md) — `get_screenshot()` compares against `get_draw_stage()`.
- [README.md](README.md) — `threading-model` config variable and its experimental-use warning.
