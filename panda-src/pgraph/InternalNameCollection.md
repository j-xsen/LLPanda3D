# InternalNameCollection

**Source:** `panda/src/pgraph/internalNameCollection.h` (+ `.I`, `.cxx`)
**Inherits:** (standalone value type)

An ordered, searchable, copy-on-write list of `InternalName` pointers.
`InternalName` (interned column/attribute name identifiers used e.g. for
vertex data columns and shader input names) is defined outside this module
(`panda/src/putil`, undocumented) — this collection just aggregates them,
same shape as [MaterialCollection](MaterialCollection.md)/
[TextureStageCollection](TextureStageCollection.md).

## Behavior notes

- Same copy-on-write-on-mutate array sharing pattern as the sibling
  collection classes (see [MaterialCollection](MaterialCollection.md)'s
  behavior notes — identical implementation, just `CPT(InternalName)`
  instead of `PT(Material)`).
- No `find_name(name)` lookup-by-string method exists here (unlike
  `MaterialCollection::find_material()`/`TextureStageCollection::
  find_texture_stage()`) — since `InternalName` instances are themselves
  interned by name, callers typically look up `InternalName::make(name)`
  directly rather than searching a collection by string.

## API

| Method | Notes |
|---|---|
| `add_name(const InternalName *)` | Append |
| `remove_name(const InternalName *)` → `bool` | |
| `add_names_from(other)` / `remove_names_from(other)` | Bulk append/subtract |
| `remove_duplicate_names()` | Keeps first occurrence |
| `has_name(const InternalName *)` → `bool` | |
| `clear()` | |
| `get_num_names()` / `size()` → `int` | Equivalent |
| `get_name(int)` / `operator[]` → `const InternalName *` | |
| `operator +`, `operator +=` | Concatenation |
| `output(ostream&)` / `write(ostream&, indent)` | |

## Usage

```cpp
InternalNameCollection names;
names.add_name(InternalName::get_texcoord());
names.add_name(InternalName::get_color());
```

## See also

- [MaterialCollection](MaterialCollection.md), [TextureStageCollection](TextureStageCollection.md) — same collection pattern
