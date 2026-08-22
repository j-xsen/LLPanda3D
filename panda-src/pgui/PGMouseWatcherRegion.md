# PGMouseWatcherRegion

**Source:** `panda/src/pgui/pgMouseWatcherRegion.h` / `.I` (empty) / `.cxx`
**Inherits:** `MouseWatcherRegion`

Internal plumbing — the `MouseWatcherRegion` every [PGItem](PGItem.md) owns
exactly one of, created in the `PGItem` constructor and never changing
identity for that item's lifetime (`PGItem::get_region()`). You'll interact
with it only indirectly through `PGItem`'s API (`get_id()`/`set_id()` actually
read/write this object's `MouseWatcherRegion` name;
`set_suppress_flags()`/`get_suppress_flags()` pass straight through to it).

```cpp
class PGMouseWatcherRegion : public MouseWatcherRegion {
public:
  PGMouseWatcherRegion(PGItem *item);
  // overrides enter_region/exit_region/within_region/without_region/
  // press/release/keystroke/candidate/move — each simply forwards to
  // the owning PGItem's identically-named method (with background=false
  // for press/release/keystroke/candidate).
private:
  PGItem *_item;             // back-pointer; nulled by PGItem's destructor
  static int _next_index;    // used to generate the default region name "pg<N>"
};
```

The default region (and hence event-id) name is `"pg" + _next_index++` — a
simple global counter, not derived from the item's node name. This is what
`PGItem::get_id()` returns before you call `set_id()`.

## See also

[PGItem.md](PGItem.md) (owner; forwards target) · [PGMouseWatcherGroup.md](PGMouseWatcherGroup.md)
(what these regions get registered into) · [README.md](README.md)
