# MaterialPool

**Source:** `panda/src/gobj/materialPool.h` (+ `.I`, `.cxx`)
**Inherits:** (none — static singleton) **Inherited by:** (none)

"There is only one in the universe" — a process-global pool that unifies
different [Material](Material.md) pointers with equivalent *values*, so
the engine doesn't waste memory on duplicate equivalent `Material` objects
or waste GSG time switching between lighting states that are really
identical. Build a temporary `Material` describing the state you want,
call `get_material()`, and use whatever it returns instead of your
temporary.

## Behavior notes

- **This is not full interning like `RenderState`/`GeomVertexFormat`.**
  `get_material(temp)` looks up by *value* (`indirect_compare_to`, i.e.
  `Material::compare_to()`), but on a first-time value it stores your
  `temp` pointer itself as the canonical instance (keyed by a *separate
  value-copy* `new Material(*temp)`, not by `temp`) — so the very first
  caller with a given value effectively "wins" and becomes the shared
  instance everyone else's equivalent lookups return. Later, if that
  same `Material*` gets mutated in place so its value no longer matches
  the frozen key copy, the *next* `get_material()` call for the new value
  treats it as a divergence and re-adopts the (now-different) pointer as
  the new canonical value for that key slot — there's no guarantee the
  pointer returned by `get_material()` is treated as immutable by the
  pool the way `RenderState::make()` treats its output.
- **`garbage_collect()`** sweeps every pooled entry and releases it if
  either its value has diverged from the key (as above) or its refcount
  has dropped to 1 (meaning the pool itself is the only remaining owner —
  nothing else references it anymore). Not run automatically; call
  periodically if churn matters.
- `release_material()` removes one entry by value (not by pointer
  identity) using the same `indirect_compare_to` lookup; `release_all_materials()`
  clears the whole pool immediately, without the "still referenced
  elsewhere" check `garbage_collect()` does.
- All operations are protected by an internal `LightMutex` — safe to call
  from multiple threads (e.g. concurrent model loaders).

## API

| Signature | Notes |
|---|---|
| `static Material *get_material(Material *temp)` | Returns the canonical shared `Material` with `temp`'s value — may literally return `temp`. |
| `static void release_material(Material *temp)` | Remove the entry matching `temp`'s value. |
| `static void release_all_materials()` | Clear the whole pool unconditionally. |
| `static int garbage_collect()` | Sweep stale/unreferenced entries; returns count released. |
| `static void list_contents(ostream &)` | Debug dump: each pooled material + its current refcount. |
| `static void write(ostream &)` | Same as `list_contents()`, via the global instance. |

## Usage

```cpp
PT(Material) temp = new Material();
temp->set_diffuse(LColor(1, 0, 0, 1));
Material *shared = MaterialPool::get_material(temp);
node_path.set_material(shared);
```

## See also

- [Material](Material.md) — the pooled value type
- `TexturePool` — same dedup pattern for `Texture`, see
  [TexturePool.md](TexturePool.md) (another fork)
