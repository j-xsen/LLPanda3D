# ShaderPool

**Source:** `panda/src/pgraph/shaderPool.h` (+ `.I`, `.cxx`)
**Inherits:** (static singleton, all-static interface)

A `ModelPool`/`TexturePool`-style filename-keyed cache for loaded `Shader`
objects (`Shader` itself is defined in `panda/src/gobj`, undocumented).
Ensures repeated loads of the same shader source file share one `Shader`
instance instead of re-parsing/re-compiling it.

## Behavior notes

- Pure static-method facade over a hidden singleton (`get_ptr()` lazily
  constructs `_global_ptr`); the private constructor prevents direct
  instantiation.
- `load_shader()` resolves the filename against the model path
  (`VirtualFileSystem::resolve_filename` + `get_model_path()`, same search
  path `ModelPool` uses) before checking/inserting into the cache — so two
  differently-spelled paths to the same file correctly share one entry.
- Shader language is guessed from the file extension if not otherwise
  specified: `.cg`/`.sha` → Cg, `.glsl`/`.vert`/`.frag`/`.geom`/`.tesc`/
  `.tese`/`.comp` → GLSL.
- Double-checked locking pattern in `ns_load_shader()`: releases the mutex
  while the (possibly slow) `Shader::load()` disk read/compile happens, then
  re-checks the cache under the lock before inserting, so a race between
  two threads loading the same shader concurrently doesn't create two
  `Shader` objects — the second thread's load is discarded and the first
  winner is returned.
- `add_shader()` unconditionally overwrites any existing pool entry for
  that filename (no merge/refcount check).
- `garbage_collect()` only evicts entries with `get_ref_count() == 1`
  (i.e. referenced solely by the pool itself, not by any node/attrib) —
  same convention as `ModelPool`/`TexturePool`.
- All methods are internally synchronized with a `LightMutex`; safe to call
  from multiple threads.

## API

| Method | Notes |
|---|---|
| `has_shader(filename)` → `bool` | Already loaded? |
| `verify_shader(filename)` → `bool` | Loads if needed, returns success |
| `load_shader(filename)` → `CPT(Shader)` | `BLOCKING`; `nullptr` on failure |
| `add_shader(filename, Shader *)` | Insert/overwrite a pre-built shader |
| `release_shader(filename)` | Remove one entry |
| `release_all_shaders()` | Clear the whole pool |
| `garbage_collect()` → `int` | Evict refcount==1 entries, returns count evicted |
| `list_contents(ostream&)` / `write(ostream&)` | Debug dump |

## Usage

```cpp
CPT(Shader) sh = ShaderPool::load_shader("myshader.glsl");
if (sh == nullptr) {
  // load failed
}
```

## See also

- [ShaderInput](ShaderInput.md) — individual values bound into a shader
- `Shader` — `panda/src/gobj/shader.h` (undocumented)
