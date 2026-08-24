# SparseArray

**Source:** `panda/src/putil/sparseArray.h` / `.I` / `.cxx`
**Inherits:** none (value type; registers a `TypeHandle` but not a `TypedObject`)

Records a set of integers — each integer either present or absent — using
the same conceptual API as [BitArray](BitArray.md) and [BitMask](BitMask.md)
(`get_bit`, `set_range`, `get_num_on_bits`, bitwise operators, ...), but a
completely different representation: a sorted list of non-overlapping
`[begin, end)` **subranges** (`_subranges`, an `ov_set<Subrange>`) plus one
`_inverse` flag, rather than a bitmask. That makes it efficient for sets
that are large *contiguous* runs (e.g. "every path index from 1000 to
50000") regardless of how sparse or huge the numeric range is, and — unlike
`BitArray` — it can represent **negative** indices. It's correspondingly bad
at sets with many isolated, non-contiguous bits, and boolean ops (`&`, `|`,
`^`) are the most expensive operation this class offers (each has to
walk/merge two subrange lists), the opposite performance profile from
`BitMask`/`BitArray` where those are single machine-word ops.

## Behavior notes

- **`_inverse` is the "is everything above the last subrange on?" flag**,
  analogous to `BitArray::_highest_bits`. `all_on()` is `_inverse = true`
  with zero subranges; `all_off()` is the default-constructed empty state.
  `is_inverse() const` exposes it directly.
- **`get_num_bits()` returns the end of the last subrange**, i.e. the
  current "interesting" range boundary — like `BitArray::get_num_bits()`,
  not a true bit count; every bit past it has the same value
  (`get_highest_bits()`).
- **`has_max_num_bits()` always returns `false`, and `get_max_num_bits()`
  always raises** (`nassert_raise`) — present purely so generic code can be
  templated across `BitMask`/`BitArray`/`SparseArray` and branch on
  `has_max_num_bits()` before calling `get_max_num_bits()`.
- **`get_bit()`/`set_bit()`/`clear_bit()` are thin wrappers around the
  range API** — `get_bit(index)` is `has_any_of(index, 1)`, `set_bit(index)`
  is `set_range(index, 1)`, etc. — so understanding `set_range`/`clear_range`
  (which merge/split subranges) explains the whole per-bit surface.
- **`get_num_on_bits()`/`get_num_off_bits()` return `-1` when infinite**,
  same convention as `BitArray` — `get_num_on_bits()` is `-1` whenever
  `_inverse` is set (infinitely many on-bits), regardless of subrange
  contents.
- **Constructible from a `BitArray`**, by walking its bits start to end and
  recording each maximal same-valued run as one subrange — an O(n) scan
  over `BitArray::get_num_bits()`, so only cheap for a `BitArray` whose
  interesting range isn't huge. There's no reverse-direction constructor;
  build a `BitArray` from a `SparseArray` via `BitArray`'s own
  `BitArray(const SparseArray&)` constructor instead (see [BitArray.md](BitArray.md)).
- **Boolean and shift operators are `O(subrange count)` merges**, not O(1)
  word ops — `do_intersection()`/`do_union()`/`do_intersection_neg()` (the
  private workers behind `&=`/`|=`/`^=`) walk both subrange lists in
  parallel; `do_shift()` (behind `<<=`/`>>=`) just adds/subtracts an offset
  to every subrange's `_begin`/`_end`, which is why shifting a `SparseArray`
  is comparatively cheap versus shifting a `BitArray`.
- **`output()` has no binary/hex form** — it prints the on-ranges directly
  (e.g. `[3, 10) [100, 105)`-style), reflecting the fact that a `SparseArray`
  isn't really a fixed-width number the way `BitMask`/`BitArray` are.
- **Bam serialization is custom**, like `BitArray` — `write_datagram()`/
  `read_datagram()` (not going through `TypedWritable`) store the subrange
  list plus the `_inverse` flag; see [BamWriter.md](BamWriter.md)/[BamReader.md](BamReader.md).

## API

Only what differs in shape from [BitMask](BitMask.md)/[BitArray](BitArray.md)
is spelled out; `get_bit`/`set_bit`/`clear_bit`/`set_bit_to`,
`has_any_of`/`has_all_of`, `set_range`/`clear_range`/`set_range_to`,
`get_lowest_on_bit`/`get_lowest_off_bit`/`get_highest_on_bit`/`get_highest_off_bit`/
`get_next_higher_different_bit`, `invert_in_place`/`has_bits_in_common`/`clear`,
`operator == != <`/`compare_to`, `operator & | ^ ~ << >>` and their `=` forms
all exist with the same names and semantics as `BitArray`.

| Signature | Notes |
|---|---|
| `SparseArray()` | All bits off |
| `SparseArray(const BitArray &from)` | Converts a `BitArray` into subrange form |
| `static SparseArray all_on()` / `all_off()` / `lower_on(int)` / `bit(int)` / `range(int, int)` | Same as `BitMask`/`BitArray` |
| `static bool has_max_num_bits()` | Always `false` |
| `static int get_max_num_bits()` | Always raises `nassert_raise` — never call it |
| `int get_num_bits() const` | End of the last subrange, not a true count |
| `bool get_highest_bits() const` | Value of every bit past the last subrange |
| `bool is_inverse() const` | Raw `_inverse` flag |
| `size_t get_num_subranges() const` | |
| `int get_subrange_begin(size_t n) const` / `int get_subrange_end(size_t n) const` | Direct subrange inspection, `[begin, end)` |

## Usage

```cpp
SparseArray visible;
visible.set_range(1000, 50000);   // one subrange, regardless of range size
visible.clear_bit(25000);          // splits it into two subranges

for (size_t i = 0; i < visible.get_num_subranges(); ++i) {
  // [visible.get_subrange_begin(i), visible.get_subrange_end(i))
}

BitArray as_bits(visible);         // convert to a bit-per-word representation
```

## See also

[BitArray.md](BitArray.md) (dense/word-based counterpart, interconverts) ·
[BitMask.md](BitMask.md) (fixed-width sibling in the same API family) ·
[README.md](README.md)
