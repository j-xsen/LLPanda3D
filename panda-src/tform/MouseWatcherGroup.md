# MouseWatcherGroup

**Source:** `panda/src/tform/mouseWatcherGroup.h` / `.cxx`
**Inherits:** [MouseWatcherBase](MouseWatcherBase.md), `ReferenceCount`

`MouseWatcherGroup` adds nothing behaviorally beyond
[MouseWatcherBase](MouseWatcherBase.md) — it exists purely so a standalone,
`PT()`-managed collection of regions can be reference-counted and attached
to (or detached from) a [MouseWatcher](MouseWatcher.md) as a unit via
`MouseWatcher::add_group()`/`remove_group()`, separately from the regions
the `MouseWatcher` owns directly through its own (inherited)
`MouseWatcherBase` interface.

## Behavior notes

- **The entire class body is an inline default constructor and standard
  type-registration boilerplate** — `mouseWatcherGroup.cxx` contains only
  the `TypeHandle` definition. All actual region-management logic lives in
  [MouseWatcherBase](MouseWatcherBase.md).
- **Grouping exists for bulk attach/detach, not for a different access
  API** — per `MouseWatcherBase`'s own doc comment, a `MouseWatcher`
  "inherits from `MouseWatcherBase`" itself, so adding regions one at a time
  directly to the `MouseWatcher` is equally valid; a separate
  `MouseWatcherGroup` is only useful when you want to later remove that
  exact batch of regions as a single operation (e.g. tearing down one UI
  panel's regions without touching any others).

## API

| Signature | Notes |
|---|---|
| `MouseWatcherGroup()` | Inline, trivial |
| (everything else) | Inherited unchanged from [MouseWatcherBase](MouseWatcherBase.md) |

## Usage

```cpp
PT(MouseWatcherGroup) panel_regions = new MouseWatcherGroup;
panel_regions->add_region(new MouseWatcherRegion("ok", LVecBase4(-0.5,0,-0.2,0.2)));
panel_regions->add_region(new MouseWatcherRegion("cancel", LVecBase4(0,0.5,-0.2,0.2)));

mouse_watcher->add_group(panel_regions);
// ... later, tear down the whole panel at once ...
mouse_watcher->remove_group(panel_regions);
```

## See also

[MouseWatcherBase](MouseWatcherBase.md) (all the actual behavior) ·
[MouseWatcher](MouseWatcher.md) (`add_group()`/`remove_group()`/
`replace_group()`)
