# SimpleHashMap

**Source:** `panda/src/putil/simpleHashMap.h` / `.I`
**Inherits:** none (standalone template)
**Inherited by:** none — but widely used as a building block by many other Panda container/index classes throughout the engine

`SimpleHashMap<Key, Value, Compare = method_hash<Key, std::less<Key>>>` is
Panda's own open-addressing hash table, used in place of `std::unordered_map`
in perf-sensitive internal code. `SimpleKeyValuePair<Key, Value>` is the
per-slot entry type; a specialization for `Value = std::nullptr_t` drops the
value storage entirely, so `SimpleHashMap<Key, std::nullptr_t>` works as a
hash *set* with zero per-entry value overhead.

## Behavior notes

- **Not an STL-compatible container.** No iterators, no `begin()`/`end()`.
  Instead you iterate by index: `for (size_t i = 0; i < map.get_num_entries(); ++i)`
  and call `get_key(i)` / `get_data(i)`. This is deliberate — see next point.
- **Entries are kept dense (index `0..size()-1` with no gaps), and `remove()`
  is safe to call *during* forward iteration by index.** Removing an entry
  moves the *last* entry into the freed slot (`_table[index] = std::move(_table[last])`)
  to avoid leaving a hole, so after removing index `n`, re-visit index `n`
  again (it now holds what used to be the last entry) rather than advancing —
  and remember the effective size just shrank by one. This is called out
  explicitly in the source ("Iterator safety" comment on `remove()`).
- **Two-array layout in one allocation.** A single block holds `_table_size`
  `TableEntry`s *followed by* a separate sparse index array of
  `_table_size * sparsity` (`sparsity = 2`) `int`s, mapping hash slot →
  dense-table index (or `-1` if empty). This is why the entry array can stay
  fully dense and cheap to iterate/copy even though the hash table itself
  (the index array) is kept sparse (2x oversized) to reduce probe-chain
  length.
- **Open addressing with linear (incrementing) probing**, wrapping via
  bitmask (`_table_size * sparsity` is always a power of two, so
  `next_hash()` is `(hash + 1) & (size - 1)`). `get_hash()` multiplies the
  `Compare`'s hash value by the odd constant `9973`, shifts, then masks —
  a cheap Knuth-style multiplicative hash.
- **`remove()` must re-thread hash-conflict chains after freeing a slot** — a
  non-trivial loop walks forward from the freed slot moving any
  displaced-by-conflict entries back toward their ideal slot, since open
  addressing correctness depends on there being no gap between an entry's
  ideal slot and its actual slot.
- **The `Compare` policy needs more than `operator()`.** Besides acting as
  the hash function (`_comp(key)` returning a `size_t`-convertible hash),
  `Compare` must also provide `is_equal(a, b)` for equality checks — this is
  a broader contract than a plain STL `Hash` functor. The default,
  `method_hash<Key, std::less<Key>>`, expects `Key` to have a `get_hash()`
  method (see `../gobj` or `dtool` for `method_hash`'s definition — it's
  outside putil).
- **Table starts empty (`_table_size == 0`, no allocation) and grows from an
  initial size of 2**, doubling (`resize_table(_table_size << 1)`) whenever
  `_num_entries == _table_size`. Shrinking (`consider_shrink_table()`) is
  implemented but not called automatically anywhere in this file — callers
  must invoke it themselves if they want a map to reclaim space after mass
  removal; tables of size ≤ 16 are never shrunk.
- **`operator[]` default-constructs `Value` on a miss** and stores it,
  exactly like `std::map::operator[]` / `std::unordered_map::operator[]`.
- **`validate()`/`write()` are O(n) consistency checkers**, useful when
  debugging hash-table corruption; `validate()` is called automatically
  after mutating operations when compiled `_DEBUG`.
- Allocation goes through `memory_hook->get_deleted_chain(alloc_size)`, the
  same small-block recycling allocator used elsewhere in Panda's low-level
  memory system — not raw `new`/`delete`.

## API

| Signature | Notes |
|---|---|
| `constexpr SimpleHashMap(const Compare& = Compare())` | Zero-allocation until first `store()` |
| `int find(const Key&) const` | Returns dense-array index, or `-1` |
| `int store(const Key&, const Value&)` | Insert or overwrite; returns index |
| `Value &operator[](const Key&)` | Find-or-default-insert |
| `bool remove(const Key&)` | See re-threading note above |
| `void remove_element(size_t n)` | Remove by dense index (looks up key, calls `remove()`) |
| `void clear()` | Deallocates everything |
| `constexpr size_t size() const` / `get_num_entries() const` | Same value, two names |
| `bool is_empty() const` | |
| `const Key &get_key(size_t n) const` / `const Value &get_data(size_t n) const` | Index-based access, `0 <= n < size()` |
| `Value &modify_data(size_t n)` | Mutable access |
| `void set_data(size_t n, const Value&)` / `(Value&&)` | |
| `bool consider_shrink_table()` | Not automatic — call manually after bulk removal |
| `void output(std::ostream&) const` / `void write(std::ostream&) const` | Debug dump; `write()` also lists every key + hash |
| `bool validate() const` | O(n) internal-consistency check |
| `void swap(SimpleHashMap&)` | O(1) pointer swap |

## Usage

```cpp
SimpleHashMap<std::string, int> counts;
counts["foo"] = 1;
counts["bar"]++;                 // operator[] default-inserts 0, then increments

for (size_t i = 0; i < counts.get_num_entries(); ++i) {
  std::cout << counts.get_key(i) << " = " << counts.get_data(i) << "\n";
}

// As a set:
SimpleHashMap<TypedObject*, std::nullptr_t> seen;
seen.store(obj, nullptr);
bool already_seen = (seen.find(obj) != -1);
```

## See also

[Comparators.md](Comparators.md) (indirect-pointer comparator helpers, a
different but related need) · [README.md](README.md)
