# Comparators

**Source:** `panda/src/putil/compareTo.h/.I`, `indirectCompareTo.h/.I`,
`indirectCompareNames.h/.I`, `indirectCompareSort.h/.I`,
`firstOfPairCompare.h/.I`, `firstOfPairLess.h/.I`
**Inherits:** none (all are standalone STL function-object templates)
**Inherited by:** none

A family of tiny STL-style comparator function objects (usable as the
`Compare` template parameter of `std::sort`, `std::set`, `pset`/`pmap`,
`SimpleHashMap`'s `Compare` policy — see [SimpleHashMap.md](SimpleHashMap.md)
— etc.). Panda passes around a lot of *pointers* to refcounted objects
rather than values, and containers of pointers sort by address by default,
which is almost never the intended order — these exist to give
pointer-keyed containers a sensible, deterministic order instead.

Each is a single-purpose one-liner; there's no shared base class or runtime
polymorphism, just `operator()`.

## The comparators

| Template | Compares | Requires | Notes |
|---|---|---|---|
| `CompareTo<ObjectType>` | **values** `a`, `b` | `a.compare_to(b)` method returning `int` | `a < b` iff `compare_to(b) < 0`. For value types, not pointers. |
| `IndirectCompareTo<ObjectType>` | **pointers** `a`, `b` | `ObjectType::compare_to(const ObjectType&)` | `a < b` iff `a != b && a->compare_to(*b) < 0`. Pointer-to-`compare_to()` version of `CompareTo`; the `a != b` short-circuit skips the virtual/member call entirely for the common "same object" case. |
| `IndirectCompareNames<ObjectType>` | **pointers**, by name | `get_name()` (e.g. any `Namable` subclass) | `a < b` iff `a != b && a->get_name() < b->get_name()`. Case-sensitive `std::string` comparison. |
| `IndirectCompareSort<ObjectType>` | **pointers**, by sort value | `get_sort()` | `a < b` iff `a->get_sort() < b->get_sort()`. **Note:** unlike the other two indirect comparators, this one does *not* short-circuit on `a == b` — calls `get_sort()` on both regardless. |
| `FirstOfPairCompare<ObjectType, Compare>` | `.first` of a pair-like `ObjectType`, via an injected `Compare` | `ObjectType` has `.first`; `Compare` is itself a comparator | Wraps another comparator (e.g. `IndirectCompareTo<Foo>`) and applies it only to `pair.first`, ignoring `.second`. Stateful — stores a `Compare _compare` instance, constructible with a custom one. |
| `FirstOfPairLess<ObjectType>` | `.first` of a pair-like `ObjectType`, via `operator<` | `ObjectType` has `.first` with `operator<` | Same idea as `FirstOfPairCompare` but hardcoded to `<` — no injected policy, so no constructor needed. |

## Which one to use

- Sorting/ordering a container of **values** with a `compare_to()`
  three-way comparison already defined → `CompareTo`.
- Sorting/ordering a container of **pointers** to objects with
  `compare_to()` (common for Panda's refcounted graph/geometry types) →
  `IndirectCompareTo`.
- Ordering by a human-readable name (e.g. a `pmap`/`pset` of
  `PT(Something)` keyed for lookup-by-name-order) → `IndirectCompareNames`.
- Ordering by an explicit numeric `sort` field (e.g. render-order-style
  sorting) → `IndirectCompareSort`.
- You have a container of `std::pair<Key, Value>` (or similar) and only
  want to compare/order by the key → `FirstOfPairLess` if plain `<` on the
  key is fine, `FirstOfPairCompare<T, SomeOtherComparator>` if the key
  itself needs a non-default comparator (e.g. pairs of pointers ordered by
  `IndirectCompareTo` on `.first`).

## Usage

```cpp
// Sort a vector of node pointers by name:
std::vector<PandaNode*> nodes = ...;
std::sort(nodes.begin(), nodes.end(), IndirectCompareNames<PandaNode>());

// A set of pairs ordered only by the (pointer) key:
std::set<std::pair<PandaNode*, int>,
         FirstOfPairCompare<std::pair<PandaNode*, int>, IndirectCompareTo<PandaNode>>> by_node;
```

## See also

[SimpleHashMap.md](SimpleHashMap.md) (its `Compare` template parameter has
a slightly broader contract — also needs `is_equal()` — these comparators
don't satisfy it directly) · [README.md](README.md)
