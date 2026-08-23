# MaterialCollection

**Source:** `panda/src/pgraph/materialCollection.h` (+ `.I`, `.cxx`)
**Inherits:** (standalone value type)

An ordered, searchable, copy-on-write list of `Material` pointers
(`Material` is defined in `panda/src/gobj`, undocumented). Typically
returned by scene-graph query methods (e.g. `NodePath::find_all_materials()`)
rather than built up by hand.

## Behavior notes

- Backed by a `PTA(PT(Material))` (a `PointerToArray`) — copies are cheap
  (shared underlying array) until a mutating call (`add_material`,
  `remove_material`) detects `get_ref_count() > 1` and copy-on-writes the
  array first, so independently-held collections never see each other's
  mutations.
- `remove_materials_from()`/`remove_duplicate_materials()` compare by
  pointer identity, not by content.
- `find_material(name)` does a linear scan comparing `get_name()`; returns
  the *first* match.
- No dedicated `MaterialPool`-style equivalent exists for `Material` in this
  module — this class is a query-result list, not a cache.

## API

| Method | Notes |
|---|---|
| `add_material(Material *)` | Append |
| `remove_material(Material *)` → `bool` | Removed / not found |
| `add_materials_from(other)` / `remove_materials_from(other)` | Set-ish bulk ops, append/subtract (no auto-dedup on add) |
| `remove_duplicate_materials()` | Keeps first occurrence of each pointer |
| `has_material(Material *)` → `bool` | |
| `clear()` | |
| `find_material(name)` → `Material *` | `nullptr` if none |
| `get_num_materials()` / `size()` → `int` | Equivalent |
| `get_material(int)` / `operator[]` → `Material *` | |
| `operator +`, `operator +=` | Concatenation |
| `output(ostream&)` / `write(ostream&, indent)` | One-line vs. multi-line dump |

## Usage

```cpp
MaterialCollection mats = some_node_path.find_all_materials();
for (int i = 0; i < mats.get_num_materials(); i++) {
  Material *m = mats.get_material(i);
}
```

## See also

- [TextureStageCollection](TextureStageCollection.md), [InternalNameCollection](InternalNameCollection.md) — same collection pattern
