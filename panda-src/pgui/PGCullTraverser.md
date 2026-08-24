# PGCullTraverser

**Source:** `panda/src/pgui/pgCullTraverser.h` / `.I` / `.cxx`
**Inherits:** `CullTraverser`

Internal plumbing — this class is almost never constructed or referenced
directly. It's a `CullTraverser` subclass that [PGTop](PGTop.md) substitutes in
for the normal traverser during its `cull_callback()`, purely to carry two
extra pieces of data through the traversal so [PGItem::cull_callback()](PGItem.md)
can register itself correctly:

```cpp
class PGCullTraverser : public CullTraverser {
public:
  INLINE PGCullTraverser(PGTop *top, CullTraverser *trav);
  PGTop *_top;        // back-pointer, so PGItems can call _top->add_region(region)
  int _sort_index;     // running counter for "unsorted" bin sort order
};
```

`_sort_index` starts at `PGTop::get_start_sort()` and is incremented once per
`PGItem` encountered whose cull bin is `BT_unsorted`, giving each item a
strictly increasing sort key equal to scene-graph traversal order.

Constructed by copying an existing `CullTraverser`'s state
(`CullTraverser(*trav)`), so it behaves identically to the traverser it
replaces except for carrying the two extra fields.

## See also

[PGTop.md](PGTop.md) (the only class that constructs one) ·
[PGItem.md](PGItem.md) (reads `_top`/`_sort_index` during its own `cull_callback`) ·
[README.md](README.md)
