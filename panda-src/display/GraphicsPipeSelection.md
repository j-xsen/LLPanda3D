# GraphicsPipeSelection

**Source:** `panda/src/display/graphicsPipeSelection.h` (+ `.I`, `.cxx`)
**Inherits:** (none — standalone class) **Inherited by:** (none)

A process-global registry and factory for the `GraphicsPipe` subclasses
available on the current platform. Concrete display-backend shared
libraries (the OpenGL module, DirectX module, TinyDisplay module, etc.)
each register their pipe type here (`add_pipe_type()`) at load/static-init
time; this class handles picking a default, matching by name, and lazily
loading the shared library ("display module") that implements a requested
type. Constructor/destructor are `protected` — only reachable through the
singleton accessor.

## Behavior notes

- **Singleton via `get_global_ptr()`**, lazily constructed. The constructor
  reads `load-display` (default `"*"`) and `aux-display` (a repeatable
  list) config variables *locally*, not from `config_display.h`, "in case
  this constructor is running at static init time" — i.e. before
  `config_display`'s own declarations are guaranteed initialized. `"*"` (or
  empty) for `load-display` means "no single preferred module — try
  everything in `aux-display`"; a non-`*` value both names the module to
  load first *and* optionally names the specific pipe class to prefer
  within it (`load-display`'s second word).
- **Module loading is deferred and incremental**, not done all at once.
  `get_num_pipe_types()`/`get_pipe_type()`/`print_pipe_types()` all call
  `load_default_module()` first (loads just the `load-display`-named module,
  or all of `aux-display` if none was named); `make_pipe(type_name)`
  escalates through four levels only as needed: already-registered → named
  module → default module → *all* aux modules (`load_aux_modules()`) —
  each level is tried only if the previous one didn't produce a match, so a
  simple call that resolves on the first try never touches the filesystem
  for the other display backends.
- **Each shared library exports a well-known symbol,
  `get_pipe_type_<module_name>`**, a C function returning an `int` — a
  `TypeHandle` id looked up via `TypeRegistry::find_type_by_id()`. This is
  how `load_named_module()` learns which `GraphicsPipe` subclass a freshly
  loaded `.so`/`.dll` recommends as its default, without the caller needing
  to already know the class name.
- **Loaded modules are cached and never reloaded** (`_loaded_modules` keyed
  by module name) — a second `make_pipe()`/`make_module_pipe()` call
  against an already-loaded module returns its cached recommended pipe type
  immediately rather than touching `load_dso()` again. If a load succeeds
  but the recommendation function is missing/returns a bogus type index, the
  module is still kept loaded (can't be safely unloaded, "because it may
  have assigned itself into the `GraphicsPipeSelection` table" per the
  source comment) — you just get a warning and `TypeHandle::none()`.
- **`make_pipe(TypeHandle)` matches exact type first, then any more-derived
  type** (`is_derived_from`), and only loads the default module and retries
  if nothing at all matched — so requesting a base type can transparently
  get you whatever more-specific subclass is actually registered.
- **`make_default_pipe()`'s name-matching is deliberately loose**: exact
  case/hyphen-underscore-insensitive match on `_default_pipe_name` first
  (from `load-display`'s second word), then falls back to a case-insensitive
  *substring* match, and only if both fail does it just take the first
  registered pipe type in the list — so a config value like `"gl"` will
  happily match `"GLGraphicsPipe"`.
- **`add_pipe_type()` guards against duplicate/wrong-hierarchy registration**
  — it refuses (returns `false`, logs a warning) if the type isn't actually
  derived from `GraphicsPipe`, or if that exact type is already registered.

## API

| Signature | Notes |
|---|---|
| `INLINE static GraphicsPipeSelection *get_global_ptr()` | Lazily-constructed singleton. |
| `int get_num_pipe_types() const` / `TypeHandle get_pipe_type(int n) const` | Triggers `load_default_module()` first. |
| `void print_pipe_types() const` | Writes known types + remaining unloaded module count to `nout`. |
| `PT(GraphicsPipe) make_pipe(const string &type_name, const string &module_name = "")` | Escalating resolution — see behavior notes. |
| `PT(GraphicsPipe) make_pipe(TypeHandle type)` | Exact match, then derived-type match. |
| `PT(GraphicsPipe) make_module_pipe(const string &module_name)` | Loads exactly the named module and constructs its recommended pipe type. |
| `PT(GraphicsPipe) make_default_pipe()` | See the name-matching behavior notes; used by `PandaFramework::make_default_pipe()` (see [../framework/PandaFramework.md](../framework/PandaFramework.md)). |
| `INLINE int get_num_aux_modules() const` | Modules still not loaded; nonzero means `load_aux_modules()` could find more pipes. |
| `void load_aux_modules()` | Loads every remaining `aux-display`-named module. |
| `bool add_pipe_type(TypeHandle type, PipeConstructorFunc *func)` (public, not `PUBLISHED`) | Registration hook called by each display backend at load time. |

## Usage

```cpp
GraphicsPipeSelection *sel = GraphicsPipeSelection::get_global_ptr();
sel->load_aux_modules();  // make every available backend visible
sel->print_pipe_types();

PT(GraphicsPipe) pipe = sel->make_pipe("wglGraphicsPipe");
if (pipe == nullptr) {
  pipe = sel->make_default_pipe();
}
```

## See also

- [GraphicsPipe.md](GraphicsPipe.md) — what this class instantiates.
- [../framework/PandaFramework.md](../framework/PandaFramework.md) — `make_default_pipe()` and `print-pipe-types` config var usage.
- [README.md](README.md) — `load-display`/`aux-display` are declared locally in this class rather than in `config_display.h`; see behavior notes above for why.
