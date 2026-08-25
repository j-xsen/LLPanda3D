# MouseWatcherBase

**Source:** `panda/src/tform/mouseWatcherBase.h` / `.I` / `.cxx`
**Inherits:** (none — plain base, deliberately not `ReferenceCount`)
**Inherited by:** [MouseWatcherGroup](MouseWatcherGroup.md), [MouseWatcher](MouseWatcher.md)

`MouseWatcherBase` is the shared implementation for "a managed, sorted
collection of [MouseWatcherRegion](MouseWatcherRegion.md)s" — factored out
so that [MouseWatcher](MouseWatcher.md) (which is also a `DataNode`) doesn't
have to inherit `ReferenceCount` twice through two different base classes.
It owns the region vector, a mutex protecting it, and (in debug builds) the
`show_regions()` wireframe-visualization machinery.

## Behavior notes

- **The region collection is a lazily-sorted `ov_set` (ordered vector),
  not a tree or hash set.** Adding a region just `push_back()`s it and sets
  `_sorted = false`; the actual `sort_unique()` (which also removes
  duplicates) is deferred until something needs sorted/deduplicated access
  — `get_num_regions()`, `get_region(n)`, or an explicit `sort_regions()`
  call. `has_region()`/`do_remove_region()` degrade to a linear scan instead
  of a binary search whenever `_sorted` is false.
- **Every public method takes `_lock` (a `LightMutex`) internally** —
  `add_region`, `has_region`, `remove_region`, `find_region`,
  `clear_regions`, `get_num_regions`, `get_region`, `show_regions`,
  `set_color`, `hide_regions`, `update_regions` all lock/unlock around their
  body. The `friend class MouseWatcher` (and `BlobWatcher`) relationship
  exists specifically so `MouseWatcher` can reach into `_regions` and reuse
  `_lock` directly for its own multi-step operations (see
  [MouseWatcher](MouseWatcher.md)'s `add_group`/`remove_group`) without a
  second, redundant locking layer.
- **`add_region()` is idempotent by contract but not by storage** — the doc
  comment says "no longer an error to call ... more than once," but nothing
  in `add_region()` itself checks for duplicates; a genuine duplicate
  pointer is only removed later, when `do_sort_regions()`'s `sort_unique()`
  runs. Until that happens, `get_num_regions()` (which forces a sort first)
  is accurate, but the raw `_regions` vector may transiently contain
  duplicates.
- **`show_regions()`/`hide_regions()`/`set_color()`/`update_regions()`
  compile to no-ops outside debug builds** (guarded by `#ifndef NDEBUG`
  inside function bodies still wrapped by `#if !defined(NDEBUG) ||
  !defined(CPPPARSER)` at the declaration level, so the symbols still exist
  for bindings/introspection) — the visualization draws region rectangles
  as `LineSegs` line loops attached under a `render2d`-relative node, purely
  for debugging, and does nothing in a release build.
- **`get_num_regions()` and `get_region(n)` are not safe against concurrent
  mutation** despite taking the lock — the doc comment on `get_region()`
  explicitly warns that another thread could remove the nth region between
  your call and using the result; the lock only protects the vector's
  internal consistency during the single call, not the caller's larger
  sequence of calls.

## API

| Signature | Notes |
|---|---|
| `void add_region(PT(MouseWatcherRegion) region)` | See idempotency caveat above |
| `bool has_region(MouseWatcherRegion *region) const` | |
| `bool remove_region(MouseWatcherRegion *region)` | Returns false if not present |
| `MouseWatcherRegion *find_region(const std::string &name) const` | First match by name; O(n) linear scan always |
| `void clear_regions()` | |
| `void sort_regions()` / `bool is_sorted() const` | Force/query the lazy sort |
| `size_t get_num_regions() const` / `MouseWatcherRegion *get_region(size_t n) const` | Force a sort first; see thread-safety caveat |
| `void output(std::ostream &) const` / `void write(std::ostream &, int indent_level = 0) const` | |
| `void show_regions(const NodePath &render2d, const std::string &bin_name, int draw_order)` | Debug-only wireframe visualization |
| `void set_color(const LColor &color)` / `void hide_regions()` / `void update_regions()` | Debug-only |

## Usage

```cpp
PT(MouseWatcherGroup) group = new MouseWatcherGroup;
group->add_region(new MouseWatcherRegion("btn", LVecBase4(-0.5, 0.5, -0.2, 0.2)));

if (group->has_region(some_region)) {
  group->remove_region(some_region);
}
```

## See also

[MouseWatcherRegion](MouseWatcherRegion.md) (the elements held) ·
[MouseWatcherGroup](MouseWatcherGroup.md) (ref-counted standalone group) ·
[MouseWatcher](MouseWatcher.md) (the `DataNode` that actually drives mouse
input against these regions)
