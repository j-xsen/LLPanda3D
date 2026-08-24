# BitArray

**Source:** `panda/src/putil/bitArray.h` / `.I` / `.cxx`
**Inherits:** none (value type; registers a `TypeHandle` but not a `TypedObject`)

A dynamically-sized bit array that behaves as if it has an **infinite**
number of bits. Where [BitMask](BitMask.md) is fixed-width and backed by one
integer word, `BitArray` grows a `PointerToArray<BitMaskNative>` of words on
demand and additionally tracks whether the conceptually-infinite tail above
its highest stored word is all-on or all-off (`_highest_bits`). Most of the
API mirrors `BitMask` exactly — same method names, same semantics for the
range actually stored — so code written against one often works against the
other with only the type changed. Convertible from a range-based
[SparseArray](SparseArray.md) via a constructor.

## Behavior notes

- **`_highest_bits` (0 or 1) is the value of every bit past the last stored
  word.** `all_on()` is represented internally as `_highest_bits = 1` with
  zero stored words, not as a huge allocated array — so `is_all_on()` /
  `is_zero()` / bitwise ops are all cheap regardless of how "large" the
  conceptual array is. `get_highest_bits() const` exposes this flag as a bool.
- **`get_num_on_bits()`/`get_num_off_bits()` return `-1` for "infinite"** —
  if `_highest_bits` is set, the on-bit count is unbounded and the method
  returns `-1` rather than looping forever (same for off-bits when
  `_highest_bits` is clear).
- **`get_highest_on_bit()`/`get_highest_off_bit()` return `-1` if the answer
  would be infinite**, i.e. `get_highest_on_bit()` returns -1 both when there
  are no on bits *and* when there are infinitely many (`_highest_bits` set) —
  callers must check `get_highest_bits()`/`is_all_on()` separately to
  disambiguate.
- **Every mutator ends by calling `normalize()`**, which pops any trailing
  words that are entirely `_highest_bits`'s value — the stored-word array is
  always the minimal representation, so `_array.size()` alone tells you the
  "interesting" bit range (`get_num_words() * num_bits_per_word`).
- **Copy-on-write storage.** `_array` is a `PointerToArray`
  (reference-counted, shared until mutated); every non-const mutating method
  calls `copy_on_write()` first, so copying a `BitArray` is O(1) until one
  copy is modified.
- **`operator <<=`/`>>=` never lose bits on the left** (per the class's own
  doc comment) — a left shift only ever grows the word array, since there's
  no fixed top; a right shift permanently discards the low bits shifted off
  the bottom, same as `BitMask`.
- **Bitwise ops (`&=`, `|=`, `^=`) special-case when one operand's stored
  array is shorter than the other's**, using that operand's `_highest_bits`
  to decide whether to truncate, extend, or invert-and-copy the other's
  extra top words — this is the mechanism that keeps the "infinite" bits
  correct without ever materializing them (see `bitArray.cxx` if
  reimplementing similar infinite-bitset logic elsewhere).
- **`compare_to()` compares `_highest_bits` first**, then words from
  highest-order to lowest — so an infinite-tail array always sorts after a
  finite one regardless of stored-word contents.
- **`output()`/`operator<<` always render hex** (unlike `BitMask`, which
  switches to binary under 40 bits) — `output_binary()`/`output_hex()` are
  still both available directly, and prefix `...1 `/`...f ` when
  `_highest_bits` is set, to visually flag the infinite tail.
- **Bam serialization is custom**, not going through the generic
  `TypedWritable` machinery — `write_datagram()`/`read_datagram()` store the
  word array in fixed 32-bit chunks (regardless of the native word size) plus
  one trailing `_highest_bits` byte; see [BamWriter.md](BamWriter.md)/[BamReader.md](BamReader.md)
  for the datagram machinery this plugs into.

## API

Only what differs from [BitMask](BitMask.md)'s shape is called out; the full
per-bit/per-range/scanning/bitwise-op surface (`get_bit`, `set_range`,
`has_any_of`, `operator & | ^ ~`, `compare_to`, ...) is identical in name and
meaning, just operating on an unbounded index range.

| Signature | Notes |
|---|---|
| `BitArray()` | All bits off |
| `BitArray(WordType init_value)` | Low word set, rest off |
| `BitArray(const SparseArray &from)` | Converts a range-based sparse set |
| `static BitArray all_on()` / `all_off()` / `lower_on(int)` / `bit(int)` / `range(int, int)` | Same as `BitMask` |
| `constexpr static bool has_max_num_bits()` | `false` (unlike `BitMask`, which returns `true`) |
| `constexpr static int get_max_num_bits()` | `INT_MAX` |
| `size_t get_num_bits() const` | Size of the *stored* range, `num_words * num_bits_per_word` — not meaningful as "total bits," since the array is conceptually infinite |
| `bool get_highest_bits() const` | Value (on/off) of every bit past the stored range |
| `size_t get_num_words() const` / `MaskType get_word(size_t n) const` / `void set_word(size_t n, WordType)` | Direct access to backing `BitMaskNative` words |
| `int get_num_on_bits() const` / `get_num_off_bits() const` | `-1` if infinite |

Everything else — `get_bit`/`set_bit`/`clear_bit`/`set_bit_to`,
`extract`/`store`, `has_any_of`/`has_all_of`, `set_range`/`clear_range`/`set_range_to`,
`get_lowest_on_bit`/`get_lowest_off_bit`/`get_highest_on_bit`/`get_highest_off_bit`/
`get_next_higher_different_bit`, `invert_in_place`/`has_bits_in_common`/`clear`,
`operator == != <`/`compare_to`, `operator & | ^ ~ << >>` and their `=` forms,
`output`/`output_binary`/`output_hex`/`write` — match [BitMask](BitMask.md)'s
signatures and semantics.

## Usage

```cpp
BitArray flags;
flags.set_bit(3);
flags.set_range(100, 20);   // grows the backing array as needed
flags.invert_in_place();    // now finite bits are inverted AND the "infinite tail" flips on

if (flags.get_highest_bits()) {
  // every bit above the highest stored word is set
}
```

## See also

[BitMask.md](BitMask.md) (fixed-width counterpart, near-identical API) ·
[SparseArray.md](SparseArray.md) (range-based sparse set; converts to `BitArray`) ·
[BamWriter.md](BamWriter.md) / [BamReader.md](BamReader.md) (custom datagram format used by `write_datagram`/`read_datagram`) ·
[README.md](README.md)
