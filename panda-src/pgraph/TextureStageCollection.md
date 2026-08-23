# TextureStageCollection

**Source:** `panda/src/pgraph/textureStageCollection.h` (+ `.I`, `.cxx`)
**Inherits:** (standalone value type)

An ordered, searchable, copy-on-write list of `TextureStage` pointers
(`TextureStage` is defined in `panda/src/gobj`, undocumented). Same shape
as [MaterialCollection](MaterialCollection.md)/[InternalNameCollection](InternalNameCollection.md),
plus a `sort()` specific to texture stage ordering.

## Behavior notes

- Same copy-on-write-on-mutate array sharing as `MaterialCollection` (see
  that doc's behavior notes — identical implementation pattern, just for
  `TextureStage`).
- `sort()` orders entries by `TextureStage::get_sort()` ascending, via a
  private `CompareTextureStageSort` functor — matches the sort-key ordering
  used when the GSG applies multiple texture stages in a `TextureAttrib`.
- `MAKE_SEQ(get_texture_stages, ...)` exposes a Python-sequence-style
  accessor (interrogate-generated; not meaningful from C++ directly).

## API

| Method | Notes |
|---|---|
| `add_texture_stage(TextureStage *)` | Append |
| `remove_texture_stage(TextureStage *)` → `bool` | |
| `add_texture_stages_from(other)` / `remove_texture_stages_from(other)` | Bulk append/subtract |
| `remove_duplicate_texture_stages()` | Keeps first occurrence |
| `has_texture_stage(TextureStage *)` → `bool` | |
| `clear()` | |
| `find_texture_stage(name)` → `TextureStage *` | `nullptr` if none |
| `get_num_texture_stages()` / `size()` → `int` | Equivalent |
| `get_texture_stage(int)` / `operator[]` → `TextureStage *` | |
| `sort()` | Orders by `TextureStage::get_sort()` |
| `operator +`, `operator +=` | Concatenation |
| `output(ostream&)` / `write(ostream&, indent)` | |

## Usage

```cpp
TextureStageCollection stages = node_path.find_all_texture_stages();
stages.sort();
```

## See also

- [MaterialCollection](MaterialCollection.md), [InternalNameCollection](InternalNameCollection.md) — same collection pattern
