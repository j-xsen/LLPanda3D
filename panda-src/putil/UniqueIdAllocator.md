# UniqueIdAllocator

**Source:** `panda/src/putil/uniqueIdAllocator.h` / `.cxx`
**Inherits:** none
**Inherited by:** none

Manages a pool of integer IDs over a fixed inclusive range `[min, max]`,
handing them out with `allocate()` and returning them with `free()`. Freed
IDs are reused **in the order they were freed** (oldest-freed-first), via an
intrusive singly-linked free chain stored directly in a `uint32_t` array —
no separate free-list container. 4 bytes of overhead per possible ID (e.g.
10,000 IDs ≈ 40KB).

## Behavior notes

- **Oldest-first reuse is a deliberate design constraint, not an
  accident.** The header notes other implementations could pack used/free
  runs more efficiently or use a bit array, but this one specifically needs
  to track *how long* an ID has been free (oldest reused first) — useful
  when a freed ID needs a "cooldown" before reuse is safe (e.g. avoiding
  stale references from something that hasn't yet processed the free
  event).
- **The free chain is stored *in* `_table`, not alongside it.** Each table
  slot holds either `IndexAllocated` (`(uint32_t)-2`) if the ID is
  currently allocated, or the index of the *next* free ID in the chain if
  it's free (with the chain's last entry holding `IndexEnd`, `(uint32_t)-1`).
  `_next_free`/`_last_free` are the head/tail of this chain. This is why
  `allocate()`/`free()` are O(1) — but also why IDs `-1` and `-2` cannot
  themselves be valid allocatable values (irrelevant in practice since IDs
  are `uint32_t` indices offset from `_min`).
- **`allocate()` returns `IndexEnd` (`(uint32_t)-1`) on exhaustion** — check
  for this specific sentinel, not just "any large value."
- **`initial_reserve_id()` is a pre-seeding escape hatch, not meant for
  general use after allocation has started.** It marks a specific ID as
  pre-allocated (e.g. reserving IDs a prior pass already handed out) by
  splicing it out of the free chain. Because the chain has no back-pointers,
  removing an arbitrary (non-head) entry requires an O(n) scan to find the
  chain predecessor — the doc comment explicitly recommends calling this
  only *before* any `allocate()` calls, and in **descending** order by id,
  which keeps the scan cheap (near O(1) amortized) by exploiting that the
  chain is still mostly in its initial `_table[i] = i+1` order. Calling it
  at other times still works, just potentially O(n) per call.
- **IDs are table-relative internally.** The public `id` is always
  `index + _min`; `free(id)` and `initial_reserve_id(id)` both assert
  `_min <= id <= _max` and convert back to `index = id - _min` before
  touching `_table`.
- **`free()` on an already-free or out-of-range id trips an assert**
  (`nassertv(_table[index] == IndexAllocated)`), it is not a silent no-op.
- **`fraction_used()`** is just `(size - free) / size` as a `PN_stdfloat`,
  handy for pool-exhaustion monitoring/logging before `allocate()` starts
  returning `IndexEnd`.
- Logs through the `uniqueIdAllocator` Notify category at `debug` (every
  allocate/free/construct/destruct call) — noisy at debug verbosity, worth
  knowing if diagnosing ID-related weirdness.

## API

| Signature | Notes |
|---|---|
| `explicit UniqueIdAllocator(uint32_t min = 0, uint32_t max = 20)` | Range is inclusive both ends |
| `uint32_t allocate()` | Returns `IndexEnd` if the pool is exhausted |
| `void initial_reserve_id(uint32_t id)` | Pre-seed a reserved id; best called before any `allocate()`, in descending order |
| `void free(uint32_t id)` | Asserts the id is currently allocated |
| `PN_stdfloat fraction_used() const` | `(size - free) / size`, range `[0, 1]` |
| `void output(std::ostream&) const` / `void write(std::ostream&) const` | Debug dump; `write()` also prints the raw table and the free chain in order |
| `static const uint32_t IndexEnd` | `(uint32_t)-1` — sentinel for chain-end / "pool exhausted" |
| `static const uint32_t IndexAllocated` | `(uint32_t)-2` — sentinel marking an in-use slot |

## Usage

```cpp
UniqueIdAllocator ids(0, 999);   // 1000 IDs available

uint32_t a = ids.allocate();     // 0
uint32_t b = ids.allocate();     // 1
ids.free(a);                     // 0 goes back to the pool
uint32_t c = ids.allocate();     // 0 again — oldest-freed-first reuse

if (ids.allocate() == UniqueIdAllocator::IndexEnd) {
  // pool exhausted
}
```

## See also

[NameUniquifier.md](NameUniquifier.md) (a different kind of "make it
unique" — strings, not recyclable integers) · [README.md](README.md)
