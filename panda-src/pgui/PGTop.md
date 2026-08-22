# PGTop

**Source:** `panda/src/pgui/pgTop.h` / `.I` / `.cxx`
**Inherits:** `PandaNode`

The root node of a PG hierarchy. Every [PGItem](PGItem.md) must be parented
somewhere below a `PGTop` node to receive mouse/keyboard events — items outside
a `PGTop` subtree still render but are never clickable. In a standard
`ShowBase` setup, `aspect2d` (and `render2d`/`pixel2d` if used for GUI) is
already a `PGTop`; you only construct one directly when building a scene graph
by hand.

## Behavior notes

- **`PGTop` has an infinite bounding volume and disables state sorting.** The
  constructor sets `OmniBoundingVolume`, `set_final(true)`, and a
  `CullBinAttrib("unsorted", 0)` — children render in scene-graph order by
  default, appropriate for 2-D UI, and nothing under `PGTop` gets culled by
  bounds.
- **Each `cull_callback()` swaps in a fresh [PGCullTraverser](PGCullTraverser.md)** and
  a fresh [PGMouseWatcherGroup](PGMouseWatcherGroup.md), traverses the subtree
  (which causes every visible `PGItem::cull_callback()` to call
  `activate_region()` and register itself into the group via `add_region()`),
  then swaps the new group into the `MouseWatcher` in place of the old one.
  This means **clickable regions are entirely rebuilt every frame** from
  whatever is actually visible — hidden or culled items are automatically not
  clickable, with no manual bookkeeping required.
- **A `MouseWatcher` must be set before items become clickable.**
  `set_mouse_watcher(nullptr)` (including via destruction) fully detaches: the
  group is removed from the watcher and destroyed.
- **`start_sort`** offsets the sort value assigned to the first `PGItem`
  discovered during traversal (subsequent items get consecutively higher
  values), which controls click priority ordering among overlapping regions
  independent of a second `PGTop`'s items. Leave at the default `0` unless you
  have multiple `PGTop`s that need a specific relative click order.

## API

### Construction
| Signature | Notes |
|---|---|
| `explicit PGTop(const std::string &name)` | |

### MouseWatcher wiring
| Signature | Notes |
|---|---|
| `void set_mouse_watcher(MouseWatcher *watcher)` | Must be called before descendant `PGItem`s are clickable; pass `nullptr` to detach |
| `MouseWatcher *get_mouse_watcher() const` | |
| `MouseWatcherGroup *get_group() const` | The current frame's region group (recreated every `cull_callback`) |
| `void set_start_sort(int)` / `int get_start_sort() const` | See "Behavior notes" |

## Usage

```cpp
PT(PGTop) top = new PGTop("gui_root");
NodePath top_np = some_2d_root.attach_new_node(top);
top->set_mouse_watcher(base_mouse_watcher);   // e.g. the app's MouseWatcher node

PT(PGButton) btn = new PGButton("ok");
btn->setup("OK");
top_np.attach_new_node(btn);   // now clickable
```

## See also

[PGItem.md](PGItem.md) (must live under a `PGTop`) ·
[PGCullTraverser.md](PGCullTraverser.md) · [PGMouseWatcherGroup.md](PGMouseWatcherGroup.md) ·
[README.md](README.md)
