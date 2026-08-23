# NodePathCollection

**Source:** `panda/src/pgraph/nodePathCollection.h` (+ `.I`, `.cxx`)

An ordered set of zero or more [NodePath](NodePath.md)s — the return type
for anything that can produce multiple paths at once
(`NodePath::get_children()`, `NodePath::find_all_matches()`, etc.) and a
convenient way to batch-apply an operation to many `NodePath`s. Most of its
API mirrors `NodePath`'s own convenience methods (`set_texture`,
`set_color`, `set_color_scale`, `reparent_to`, `show`/`hide`/`stash`, `set_attrib`,
`set_collide_mask`, `calc_tight_bounds`, …), applied to every element.

`nodePathCollection_ext.h/.cxx` (Python `__init__`/`__reduce__`/sequence
protocol glue) is excluded — Python-only.

## API

| Method | Notes |
|---|---|
| `add_path` / `remove_path` / `has_path` | Single-element membership ops |
| `add_paths_from` / `remove_paths_from` / `operator+=` / `operator+` | Combine with another collection |
| `remove_duplicate_paths()` | Dedupe by `NodePath` equality |
| `clear()` / `reserve(n)` | |
| `is_empty()` / `get_num_paths()` / `get_path(i)` / `operator[]` / `size()` | Access |
| `find_all_matches(path)` | Runs `NodePath::find_all_matches` from every element, concatenating results |
| `reparent_to(other)` / `wrt_reparent_to(other)` | Batch reparent |
| `show()` / `hide()` / `stash()` / `unstash()` / `detach()` | Batch visibility/graph ops |
| `get_collide_mask()` / `set_collide_mask(...)` | Batch collide-mask read/write |
| `calc_tight_bounds(min, max)` | Combined tight bounds across all elements |
| `set_texture(...)` / `set_texture_off(...)` | Batch texture state |
| `set_color(...)` / `set_color_scale(...)` / `compose_color_scale(...)` | Batch color state |
| `set_attrib(attrib, priority)` | Apply an arbitrary `RenderAttrib` to every element |
| `ls()` / `write()` | Debug dump |

## Usage

```cpp
NodePath scene("scene");
NodePathCollection lights = scene.find_all_matches("**/=light");
lights.set_color_scale(0.5, 0.5, 0.5, 1.0);
```

## See also

[NodePath](NodePath.md), [FindApproxPath](FindApproxPath.md)
