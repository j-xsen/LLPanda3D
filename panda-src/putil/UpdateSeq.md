# UpdateSeq

**Source:** `panda/src/putil/updateSeq.h` / `.I` / `.cxx` (`.cxx` is empty — everything is inline)
**Inherits:** none (small value type, wraps one atomic integer)
**Inherited by:** none — used as a member/return type all over the engine
(e.g. [`TypedWritable::get_bam_modified()`](TypedWritable.md), geom/vertex
cache-validity checks elsewhere) wherever code needs a cheap "has this
changed since I last looked?" timestamp

A monotonically-increasing sequence number used as a lightweight
change-timestamp: store an `UpdateSeq` alongside cached data, bump a
"current" `UpdateSeq` whenever the source changes, and compare
(`cached_seq < current_seq`) to know whether the cache is stale. Wraps a
single `AtomicAdjust::Integer`, so comparisons and increments are cheap and
`operator++` is safe to call from multiple threads without external locking.

## Behavior notes

- **Three special, non-numeric-feeling values, encoded as reserved integers:**
  `initial()` (`0`), `old()` (`1`), and `fresh()` (all-bits-set, i.e. the
  maximum unsigned value). A default-constructed `UpdateSeq` is
  `initial()`. Ordering treats these specially: **`initial` is older than
  everything, `old` is older than everything except `initial`, `fresh` is
  newer than everything** — comparisons involving any special value fall
  back to a plain unsigned compare of the underlying integer (`0 < 1 < ~0u`
  happens to already encode exactly this ordering).
- **Ordinary (non-special) values compare via *circular* signed-difference
  arithmetic**, not a plain `<`: `priv_lt(a, b)` is
  `(signed int)(a - b) < 0`. This is deliberate wraparound-tolerant
  comparison (the same trick TCP sequence numbers use) — after a sequence
  wraps around from near `UINT_MAX` back to small numbers, values "close
  after" a wrap still compare correctly as newer, as long as the two values
  being compared aren't more than half the integer range apart.
- **Incrementing (`operator++`, both prefix and postfix) never lands on a
  special-value bit pattern.** If incrementing would produce a value that
  collides with `initial`/`old`/`fresh`'s reserved bit patterns (this
  happens once, on wraparound past `SC_fresh` back to `0`), the new value
  is forced to `SC_old + 1` (`2`) instead of `0`, skipping past the
  special-value zone. This is why you should never assume "the sequence
  wrapped to exactly 0" — it wraps to 2.
- **Thread-safe increment via CAS loop.** When `HAVE_THREADS` is defined,
  `operator++` uses `AtomicAdjust::compare_and_exchange()` in a retry loop
  rather than a plain read-modify-write, so concurrent `++seq` calls from
  multiple threads never lose an increment or double-apply the
  special-value-skip logic.
- **`is_special()` is a single branchless comparison**: `(seq + 1) <= 2`
  (unsigned) — true only for `seq` in `{~0u, 0, 1}` i.e. `{fresh, initial,
  old}`, exploiting that `~0u + 1 == 0` wraps.
- **Copying is atomic-load/atomic-store (`AtomicAdjust::get`/`::set`), not a
  trivial memcpy** — safe to copy an `UpdateSeq` that another thread might
  concurrently be incrementing.
- **`get_seq()` is documented "useful for debugging only"** — don't build
  logic on the raw integer value; use the comparison operators and
  `is_initial()`/`is_old()`/`is_fresh()`/`is_special()` instead, since the
  special-value encoding and wraparound-skip behavior are meant to stay
  opaque.

## API

| Signature | Notes |
|---|---|
| `constexpr UpdateSeq()` | Starts as `initial()` |
| `static UpdateSeq initial()` / `old()` / `fresh()` | The three special sentinel values |
| `void clear()` | Resets to `initial()` |
| `bool is_initial() const` / `is_old() const` / `is_fresh() const` / `is_special() const` | |
| `operator==`, `!=`, `<`, `<=`, `>`, `>=` | Special-value-aware, wraparound-tolerant on ordinary values |
| `UpdateSeq operator++()` / `operator++(int)` | Thread-safe; skips special-value bit patterns on wraparound |
| `AtomicAdjust::Integer get_seq() const` | Raw value — debugging only |
| `void output(std::ostream&) const` | Prints `"initial"`/`"old"`/`"fresh"` or the numeric value |

## Usage

```cpp
class Cache {
  UpdateSeq _cached_as_of = UpdateSeq::initial();  // "never computed"
  UpdateSeq _source_seq;                            // bumped whenever source changes

  void mark_source_dirty() { ++_source_seq; }

  const Data &get() {
    if (_cached_as_of < _source_seq) {
      recompute();
      _cached_as_of = _source_seq;
    }
    return _data;
  }
};
```

## See also

[TypedWritable.md](TypedWritable.md) (`get_bam_modified()` returns an
`UpdateSeq`) · [README.md](README.md)
