# PGMouseWatcherGroup

**Source:** `panda/src/pgui/pgMouseWatcherGroup.h` / `.I` / `.cxx`
**Inherits:** `MouseWatcherGroup`

Internal plumbing — the `MouseWatcherGroup` each [PGTop](PGTop.md) owns.
Recreated fresh every frame by `PGTop::cull_callback()` — see
[PGTop.md](PGTop.md) for why. Exists as its own tiny subclass — rather than
`PGTop` directly inheriting `MouseWatcherGroup` — to avoid a
circular-reference-count problem between `NamedNode` and `MouseWatcherGroup`
(per the header comment).

```cpp
class PGMouseWatcherGroup : public MouseWatcherGroup {
public:
  INLINE PGMouseWatcherGroup(PGTop *top);
  INLINE void clear_top(PGTop *top);   // called by PGTop when it drops this pointer
private:
  PGTop *_top;
};
```

**Destructor behavior worth knowing:** if a `PGMouseWatcherGroup` is destroyed
while `_top` is still set (i.e. its `PGTop` didn't call `clear_top()` first —
shouldn't normally happen in the intended lifecycle, but is defensively
handled), it reaches back into the `PGTop` and forces
`_top->_watcher_group = nullptr; _top->set_mouse_watcher(nullptr);`, fully
detaching the `PGTop` from its `MouseWatcher`.

## See also

[PGTop.md](PGTop.md) (owner) · [PGMouseWatcherRegion.md](PGMouseWatcherRegion.md)
(what gets added to this group) · [README.md](README.md)
