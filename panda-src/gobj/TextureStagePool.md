# TextureStagePool

**Source:** `panda/src/gobj/textureStagePool.h` (+ `.I`, `.cxx`)
**Inherits:** (none — static singleton, private constructor)

A single global pool (`get_global_ptr()`, private constructor — never
instantiate directly) that unifies distinct `TextureStage` pointers with
equivalent properties, so different model files that each define "their
own" e.g. `"normal"` stage end up sharing one C++ object after loading. All
methods are static wrappers around a private, mutex-guarded (`Mutex _lock`)
singleton implementation, so it's thread-safe to call from multiple loader
threads.

## Behavior notes

- Operates in one of three `Mode`s, set via `set_mode()` (or the
  `texture-stage-pool-mode` config var, default `M_none`):
  - `M_none` — pool disabled; `get_stage()` is a no-op passthrough.
  - `M_name` — stages are unified by `get_name()` only (a `pmap<string,
    PT(TextureStage)>`); the *first* stage registered under a given name
    wins and is returned for all later `get_stage()` calls with that name,
    regardless of whether other properties differ.
  - `M_unique` — stages are unified by full value equality (via
    `TextureStage::compare_to()`, same as `operator==`), keyed by a
    `CPT(TextureStage)` acting as a lookup key with an
    `indirect_compare_to` comparator; only truly-identical stages merge.
- Switching mode via `set_mode()` clears both internal maps immediately —
  changing mode does not migrate or reconcile existing pooled entries.
- `get_stage()`/`release_stage()` are called automatically by the loader
  pipeline (e.g. egg/bam loaders) when reading stored `TextureStage`s off
  disk — app code rarely calls this pool directly.
- `garbage_collect()` sweeps entries whose backing `TextureStage` either
  has `get_ref_count() == 1` (nothing but the pool itself holds it anymore)
  or whose live key no longer matches the map key (name changed after
  pooling, in `M_name` mode) — both cases are removed. Returns the count
  released. Not run automatically; something (e.g. a periodic task) must
  call it.
- `list_contents()`/`write()` dump the pool for debugging, including each
  entry's current refcount.

## API

| Signature | Notes |
|---|---|
| `enum Mode { M_none, M_name, M_unique }` | See behavior notes. |
| `static TextureStage *get_stage(TextureStage *temp)` | Look up/register `temp`; returns the pooled (possibly different) pointer. |
| `static void release_stage(TextureStage *temp)` | Remove `temp` from the pool. |
| `static void release_all_stages()` | Clear both internal maps. |
| `static void set_mode(Mode) / static Mode get_mode()` | Also `MAKE_PROPERTY(mode, ...)`. Switching mode clears the pool. |
| `static int garbage_collect()` | Sweep stale/orphaned entries; returns count released. |
| `static void list_contents(ostream&)` / `static void write(ostream&)` | Debug dump. |

## Usage

```cpp
PT(TextureStage) ts = new TextureStage("normal");
ts = TextureStagePool::get_stage(ts); // may return a shared existing instance
```

## See also

- [TextureStage](TextureStage.md)
