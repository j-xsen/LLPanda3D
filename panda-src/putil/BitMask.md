# BitMask / DoubleBitMask

**Source:** `panda/src/putil/bitMask.h` / `.I` (template `BitMask<WType, nbits>`) +
`panda/src/putil/doubleBitMask.h` / `.I` (template `DoubleBitMask<BMType>`) +
`drawMask.h`, `collideMask.h`, `portalMask.h` (typedef aliases) +
`pbitops.h` (internal bit-scan helpers, mentioned below)
**Inherits:** none (standalone value types; `get_class_type()`/`init_type()` register a `TypeHandle` but neither derives from `TypedObject`)

`BitMask<WType, nbits>` is a **fixed-width** bitmask backed by a single
unsigned integer word (`WType`) — `nbits` must fit in `WType`'s bit width.
It's the workhorse type for engine bit-flag fields (collide masks, draw
masks, camera visibility masks) where the bit count is small and known at
compile time. For an unbounded/dynamically-sized bit set, see
[BitArray.md](BitArray.md); for a range-based sparse set, see
[SparseArray.md](SparseArray.md) — `BitMask`, `BitArray`, and `SparseArray`
share most of the same method names (`get_bit`, `set_range`, `get_num_on_bits`,
`operator &`, ...) so code can often be templated across all three.

`DoubleBitMask<BMType>` composes two `BMType` instances (`_lo`/`_hi`) to
present a mask twice as wide as its component type — e.g.
`DoubleBitMask<BitMaskNative>` doubles the native word size to 64 bits on a
32-bit build. It exposes the same API surface as `BitMask` and can itself be
the `BMType` of another `DoubleBitMask` to double again (`QuadBitMaskNative`).

## Behavior notes

- **All storage is a single value-type word (or pair of words for
  `DoubleBitMask`)** — `BitMask` has no dynamic allocation, virtual
  functions, or reference counting; it's cheap to copy, pass by value, and
  put in a hash key (`get_key()` just returns the word cast to `int`).
- **Bit 0 is the least-significant bit.** `bit(index)` sets bit `index`, i.e.
  `1 << index`. `output()`/`operator<<` print highest bit first (leftmost),
  matching normal binary/hex notation, and auto-switch from binary to hex
  once `num_bits >= 40` (`output_hex()` groups 4 bits/spaces by default).
- **`extract()`/`store()` treat the mask as a bitfield-packed integer** —
  `extract(low_bit, size)` returns those bits right-shifted to position 0;
  `store(value, low_bit, size)` writes `value`'s low `size` bits into that
  range, leaving the rest of the word untouched.
- **`get_next_higher_different_bit(low_bit)` is the documented way to
  iterate a mask's runs of same-valued bits** — starting from `low_bit`, it
  returns the index of the next bit whose value differs from `get_bit(low_bit)`,
  or `low_bit` itself again if no higher bit differs (or `num_bits` if the
  word is uniform above `low_bit` and `low_bit` was on). Passing `low_bit ==
  num_bits` is explicitly legal (asserts only `low_bit >= 0`).
- **`flood_bits_up()`/`flood_bits_down()`** set every bit above (resp.
  below) the lowest (resp. highest) "on" bit — used to build masks like "all
  bits from here up." Backed by `::flood_bits_up`/`::flood_bits_down` from
  `pbitops.h`.
- **`keep_next_highest_bit(index)` / `keep_next_lowest_bit(index)`** return a
  mask with only the single on-bit nearest to (but not including) `index`,
  computed via a flood/invert/AND trick rather than a scan loop — see
  `bitMask.I` if reimplementing. The overloads taking another `BitMask`
  instead of an `int` use that mask's highest/lowest on-bit as the pivot,
  falling back to the no-argument form (nearest to the mask's own
  highest/lowest bit) if the passed mask is empty.
- **`operator <` imposes an arbitrary but total order** (numeric word
  comparison) — not meaningful as "is a subset of," it exists purely so
  `BitMask` can key ordered/unordered STL containers (required to export such
  containers under Windows).
- **`DoubleBitMask` does not implement `get_word()`/`set_word()`** (no
  single word wide enough to represent it) or the shift-related
  `flood_*`/`keep_next_*` helpers `BitMask` has — its API is otherwise a
  mirror of `BitMask`'s, delegating each half's work to `_lo`/`_hi` with
  carry/borrow handled across the split (see `doubleBitMask.I` for the
  carry logic in `operator +`-style range/shift ops).
- **Every `BitMask`/`DoubleBitMask` instantiation used anywhere in the
  engine must be explicitly template-instantiated and exported**
  (`EXPORT_TEMPLATE_CLASS` in `bitMask.h`/`doubleBitMask.h`) — only
  `BitMask16`, `BitMask32`, `BitMask64`, `DoubleBitMaskNative`, and
  `QuadBitMaskNative` are pre-instantiated; a new width requires adding an
  explicit instantiation, not just a typedef.
- **`pbitops.h`** (not documented as its own class — free functions, no
  state) supplies the low-level, per-word-type-overloaded primitives
  `count_bits_in_word()`, `flood_bits_up()`/`flood_bits_down()`, and
  `get_lowest_on_bit()`/`get_highest_on_bit()`/`get_next_higher_bit()` that
  `BitMask` and `BitArray` build on; `count_bits_in_word()` for 16-bit words
  is a lookup into a precomputed 65536-entry table (`num_bits_on`), and
  wider words decompose into 16-bit chunks. There's normally no reason to
  call these directly — use the `BitMask`/`BitArray` methods instead.

## Common typedefs

| Typedef | Definition | Used for |
|---|---|---|
| `BitMask16` | `BitMask<uint16_t, 16>` | |
| `BitMask32` | `BitMask<uint32_t, 32>` | |
| `BitMask64` | `BitMask<uint64_t, 64>` | |
| `BitMaskNative` | `BitMask32` or `BitMask64` | Whichever matches `NATIVE_WORDSIZE` |
| `DoubleBitMaskNative` | `DoubleBitMask<BitMaskNative>` | |
| `QuadBitMaskNative` | `DoubleBitMask<DoubleBitMaskNative>` | |
| `DrawMask` | `BitMask32` (`drawMask.h`) | Camera/`PandaNode` visibility bits — a node must share a bit with the active camera's draw mask to render |
| `CollideMask` | `BitMask32` (`collideMask.h`) | `CollisionNode`/`GeomNode` collide bits — two nodes need a bit in common to be tested. By convention the low 20 bits default-on for `CollisionNode` (`default_collision_node_collide_mask`), bit 20 default-on for `GeomNode` (`default_geom_node_collide_mask`); bits 21-31 unassigned |
| `PortalMask` | `BitMask32` (`portalMask.h`) | Portal-culling visibility bits, same shape as `CollideMask` |

## API

### Construction / whole-mask queries
| Signature | Notes |
|---|---|
| `BitMask()` | All bits off (value-initialized) |
| `constexpr BitMask(WordType init_value)` | From a raw word |
| `static BitMask all_on()` / `all_off()` | |
| `static BitMask lower_on(int on_bits)` | Low `on_bits` bits set |
| `static BitMask bit(int index)` | Single bit set |
| `static BitMask range(int low_bit, int size)` | `size` bits set, starting at `low_bit` |
| `constexpr int get_num_bits() const` | `= nbits` (or `2*BMType::num_bits` for `DoubleBitMask`) |
| `bool is_zero() const` / `bool is_all_on() const` | |
| `WordType get_word() const` / `void set_word(WordType)` | `BitMask` only |

### Per-bit / per-range
| Signature | Notes |
|---|---|
| `bool get_bit(int index) const` | Asserts `0 <= index < num_bits` |
| `void set_bit(int index)` / `clear_bit(int index)` / `void set_bit_to(int index, bool)` | |
| `WordType extract(int low_bit, int size) const` / `void store(WordType, int low_bit, int size)` | |
| `bool has_any_of(int low_bit, int size) const` / `bool has_all_of(...) const` | |
| `void set_range(int low_bit, int size)` / `clear_range(...)` / `set_range_to(bool, ...)` | |
| `void clear()` | All bits off |
| `void invert_in_place()` | |

### Scanning
| Signature | Notes |
|---|---|
| `int get_num_on_bits() const` / `get_num_off_bits() const` | |
| `int get_lowest_on_bit() const` / `get_lowest_off_bit() const` | -1 if none |
| `int get_highest_on_bit() const` / `get_highest_off_bit() const` | -1 if none |
| `int get_next_higher_different_bit(int low_bit) const` | See behavior notes |
| `bool has_bits_in_common(const BitMask &other) const` | Equivalent to `(a & b) != 0` |

### Bit ops / comparison
| Signature | Notes |
|---|---|
| `operator & \| ^ ~` / `operator &= \|= ^=` | Bitwise, non-mutating and mutating forms |
| `operator << >> <<= >>=` (int shift) | `BitMask` only |
| `operator == != <` / `int compare_to(const BitMask&) const` | `<` is an arbitrary total order, not "subset of" |
| `int get_key() const` | Word cast to `int`, for hashing |

### BitMask-only extras
| Signature | Notes |
|---|---|
| `BitMask flood_bits_up() const` / `flood_bits_down() const` / `void flood_up_in_place()` / `flood_down_in_place()` | |
| `BitMask keep_next_highest_bit([int index \| const BitMask &other]) const` / `keep_next_lowest_bit(...)` | Single-bit mask nearest to `index`/the other mask's extreme on-bit |

### Output
| Signature | Notes |
|---|---|
| `void output(std::ostream&) const` | Auto binary (`<40` bits) or hex |
| `void output_binary(std::ostream&, int spaces_every = 4) const` / `output_hex(...)` | |
| `void write(std::ostream&, int indent_level = 0) const` | |
| `operator <<` (free function) | Calls `output()` |

## Usage

```cpp
CollideMask into_mask = default_collision_node_collide_mask;
CollideMask from_mask = CollideMask::bit(20) | CollideMask::bit(5);

if (into_mask.has_bits_in_common(from_mask)) {
  // these two nodes are eligible to collide
}

BitMask32 flags;
flags.set_range(4, 3);         // bits 4,5,6 on
int n = flags.get_num_on_bits(); // 3
std::cout << flags;            // prints as binary (fewer than 40 bits)
```

## See also

[BitArray.md](BitArray.md) (unbounded dynamic bit set, same API shape) ·
[SparseArray.md](SparseArray.md) (range-based sparse set) · [README.md](README.md)
