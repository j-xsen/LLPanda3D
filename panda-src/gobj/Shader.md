# Shader

**Source:** `panda/src/gobj/shader.h` (+ `.I`, `.cxx`)
**Inherits:** TypedWritableReferenceCount **Inherited by:** (none)

A compiled shader program — vertex/fragment/geometry/tessellation/compute
stages, in Cg, GLSL, HLSL, or SPIR-V. You load one with `Shader::load()`
(one file per language variant, or separate files per stage) or build one
from in-memory source with `Shader::make()`, then apply it to geometry via
`ShaderAttrib` (`pgraph`, see [../pgraph/README.md](../pgraph/README.md))
rather than binding it directly. The GSG turns a `Shader` into a backend
program object on demand through a `ShaderContext` — see
[ShaderContext.md](ShaderContext.md) and the module README's
"`PreparedGraphicsObjects` / `*Context` handshake" section for that
pattern.

## Behavior notes

- **Two-level caching.** `Shader::load()` keys a static `_load_table` by
  the exact set of filenames (+ language) requested — a second `load()`
  call with the same filenames returns the same `Shader` instance,
  provided `check_modified()` (mtimes of the source + all `#include`d
  files) says nothing changed on disk; a stale hit triggers a full
  reload. Independently, if `cache-generated-shaders` is set, a second
  static `_make_table` keyed by the *resolved shader body text* means two
  different `load()`/`make()` calls that happen to produce identical
  source (e.g. two different filenames whose contents are byte-identical,
  or a `load()` and a `make()` with matching text) collapse to one shared
  `Shader` object. `load_compute()` additionally consults the on-disk
  `BamCache` (extension `"sho"`) before recompiling.
- **`prepare()`/`prepare_now()`/`release()`/`release_all()`** mirror
  `Texture`'s lifecycle exactly: `prepare()` enqueues async preparation on
  a `PreparedGraphicsObjects`, `prepare_now()` compiles synchronously and
  is normally only called by the GSG itself, `release()` frees the
  `ShaderContext` on one `PreparedGraphicsObjects`, `release_all()` on
  every one the shader is prepared on. A `Shader` can be simultaneously
  prepared against multiple GSGs (`_contexts` is a
  `PreparedGraphicsObjects* → ShaderContext*` map).
- **Compiled-binary caching is separate from source caching.**
  `set_compiled()`/`get_compiled()` let a GSG backend stash its own
  compiled binary blob (format tag + bytes) on the `Shader`, persisted via
  `_record` (a `BamCacheRecord`) when `cache-compiled-shader` is set —
  this avoids recompiling GLSL/HLSL source on every process run even
  when the disk-cache source lookup above misses.
- **Input binding is name/convention-driven, not manually wired.** The
  `cp_*`/`compile_parameter()` machinery (invoked once per shader, at
  `prepare_now()` time) parses each shader parameter's declared name
  against a large fixed vocabulary (`ShaderMatInput` — e.g.
  `p3d_ModelViewMatrix`-style built-ins for transforms/lights/fog/clip
  planes; `ShaderTexInput` for texture-stage-derived samplers) and records
  how to refresh it into one of `_mat_spec`/`_tex_spec`/`_var_spec`/
  `_ptr_spec`. The GSG re-evaluates these specs against the current
  `RenderState` every time the effective state changes (see
  `ShaderStateDep` bits — `SSD_transform`, `SSD_light`, `SSD_fog`, …
  gate which specs need re-evaluation), rather than the application
  pushing uniform values explicitly for built-in inputs. Custom
  (non-built-in) uniforms come from `ShaderInput`s set on the
  `ShaderAttrib` and are matched by name via `_var_spec`/`_ptr_spec`.
- **GLSL preprocessing is Panda's own**, not the driver's: when
  `glsl-preprocess` is set, `r_preprocess_source()`/
  `r_preprocess_include()` recursively resolve `#include`/`#pragma` in
  GLSL source text before handing it to the driver, capped by
  `glsl-include-recursion-limit`; included files are tracked in
  `_included_files` so `check_modified()` can detect changes to them too,
  and are addressable via `get_filename_from_index()` (indices ≥ 2048
  encode "the Nth included file", used for mapping compiler error
  line/file numbers back to source).
- The `parse_*`/`cp_*`/`cg_*` members are internal to the Cg-era text-based
  parameter parser and GSG backend compile paths (`public:` for
  cross-TU access from GSG implementations, not meant to be called by
  application code) — omitted from the API table below as internal
  plumbing, not real public surface.

## API

**Loading:**

| Signature | Notes |
|---|---|
| `static PT(Shader) load(Filename file, ShaderLanguage lang = SL_none)` | Single-file shader (language inferred from content/extension if `SL_none`). |
| `static PT(Shader) load(ShaderLanguage lang, Filename vertex, Filename fragment, Filename geometry = "", Filename tess_control = "", Filename tess_evaluation = "")` | Separate file per stage. |
| `static PT(Shader) load_compute(ShaderLanguage lang, Filename fn)` | GLSL-only; compute shaders are always separate/single-stage. |
| `static PT(Shader) make(string body, ShaderLanguage lang = SL_none)` | In-memory source; not valid for GLSL (needs separate stage bodies). |
| `static PT(Shader) make(ShaderLanguage lang, string vertex, string fragment, string geometry = "", string tess_control = "", string tess_evaluation = "")` | In-memory, per-stage. |
| `static PT(Shader) make_compute(ShaderLanguage lang, string body)` | In-memory compute shader (GLSL only). |

**Introspection:**

| Signature | Notes |
|---|---|
| `Filename get_filename(ShaderType type = ST_none) const` | Per-stage source filename (`ST_none` = the shared/first one). |
| `const string &get_text(ShaderType type = ST_none) const` | Resolved source text for a stage. |
| `bool get_error_flag() const` | Set if loading/compiling ever failed. |
| `ShaderLanguage get_language() const` | `SL_none/Cg/GLSL/HLSL/SPIR_V`. |
| `bool has_fullpath() const` / `const Filename &get_fullpath() const` | Resolved on-disk path, if loaded from a file. |
| `bool get_cache_compiled_shader() const` / `void set_cache_compiled_shader(bool)` | Whether to persist the compiled binary via `BamCacheRecord`. |

**GSG preparation:**

| Signature | Notes |
|---|---|
| `PT(AsyncFuture) prepare(PreparedGraphicsObjects *)` | Queue async compile. |
| `bool is_prepared(PreparedGraphicsObjects *) const` | Already compiled or queued? |
| `bool release(PreparedGraphicsObjects *)` | Free on one GSG. |
| `int release_all()` | Free on every GSG; returns count freed. |
| `ShaderContext *prepare_now(PreparedGraphicsObjects *, GraphicsStateGuardianBase *)` | Synchronous compile; normally GSG-internal only. |

## Usage

```cpp
PT(Shader) shader = Shader::load(
  Shader::SL_GLSL,
  "myshader.vert", "myshader.frag");
if (shader != nullptr) {
  node_path.set_shader(shader);
}
```

## See also

- [ShaderContext](ShaderContext.md) — the GSG-side compiled handle this
  class prepares
- [ShaderBuffer](ShaderBuffer.md), [SamplerState](SamplerState.md) —
  other shader-input-adjacent resources
- `ShaderAttrib`, `ShaderInput` — `pgraph`
  ([../pgraph/README.md](../pgraph/README.md)), the state-pipeline objects
  application code actually uses to attach a `Shader` and its custom
  inputs to geometry
