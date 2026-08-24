# Vectors and PTAs (explicit instantiation headers)

**Source:** `panda/src/putil/vector_typedWritable.h/.cxx`,
`vector_writable.h/.cxx`, `vector_ulong.h/.cxx`, `vector_ushort.h/.cxx`,
`pta_ushort.h/.cxx`, `ioPtaDatagramFloat.h/.cxx`, `ioPtaDatagramInt.h/.cxx`,
`ioPtaDatagramShort.h/.cxx`
**Inherits/Inherited by:** n/a — these are typedef/instantiation headers, not class hierarchies

None of these files define real new classes with behavior — they're all
one of two patterns Panda uses to avoid every DLL/shared-library that needs
`std::vector<T>` or `PointerToArray<T>` of some common `T` from separately
(and possibly inconsistently) instantiating the template and exporting
symbols for it.

## Pattern 1: `vector_X` — one canonical `std::vector<T>` instantiation

`vector_typedWritable.h`, `vector_writable.h`, `vector_ulong.h`,
`vector_ushort.h` all follow the same macro-based template: define
`EXPCL`/`EXPTP` (DLL export macros), `TYPE` (the element type), and `NAME`
(the resulting typedef name), then `#include "vector_src.h"` (declaration)
/ `"vector_src.cxx"` (explicit instantiation), both defined outside putil
(shared boilerplate every `vector_X.h` in the engine uses). The net effect
is a single exported instantiation of `pvector<TYPE>` named `NAME`, with
comments in each header explicitly telling other code: *"this class is
defined once here ... other packages ... should include this header file,
rather than defining the vector again."*

| Header | Typedef produced | Element type |
|---|---|---|
| `vector_typedWritable.h` | `vector_typedWritable` | `TypedWritable*` — link [TypedWritable.md](TypedWritable.md) |
| `vector_writable.h` | `vector_writable` | `Writable*` (a base class defined outside putil) |
| `vector_ulong.h` | `vector_ulong` | `unsigned long` |
| `vector_ushort.h` | `vector_ushort` | `unsigned short` |

There is no behavior to document beyond "it's `pvector<TYPE>` under a
shared, DLL-exported name" — treat these purely as canonical typedefs to
reuse rather than redeclaring `pvector<unsigned short>` yourself elsewhere.

## Pattern 2: `pta_X` — one canonical `PointerToArray<T>` instantiation

`pta_ushort.h` is the putil-local example of this pattern (most other
`pta_X` typedefs — `pta_int`, `pta_stdfloat`, etc. — live outside putil but
follow the identical shape). It explicitly instantiates and DLL-exports the
full `PointerToArray<T>` template chain
(`PointerToBase<ReferenceCountedVector<T>>` →
`PointerToArrayBase<T>` → `PointerToArray<T>` /
`ConstPointerToArray<T>`) for one element type, and defines the
conventional `PTA_x`/`CPTA_x` typedef pair:

```cpp
typedef PointerToArray<unsigned short> PTA_ushort;
typedef ConstPointerToArray<unsigned short> CPTA_ushort;
```

`PointerToArray<T>`/`ConstPointerToArray<T>` themselves (refcounted,
copy-on-write-free shared arrays — Panda's alternative to
`std::shared_ptr<std::vector<T>>`) are defined outside putil
(`pointerToArray.h` in `dtool`/elsewhere) — not documented here.

## Pattern 3: `IoPtaDatagram{Float,Int,Short}` — Bam I/O for a `PTA_x`

`ioPtaDatagramFloat.h/.cxx`, `ioPtaDatagramInt.h/.cxx`,
`ioPtaDatagramShort.h/.cxx` each define a tiny never-instantiated class
(`IoPtaDatagramFloat`/`Int`/`Short`, aliased `IPD_float`/`IPD_int`/`IPD_ushort`)
that exists only to scope two static free functions — the header says
outright "It's not intended to be constructed." These implement the
recurring "serialize a `PTA_x` as `[uint32 count][count * elements]`" I/O
pattern used when some other `TypedWritable`'s `write_datagram()`/`fillin()`
(see [TypedWritable.md](TypedWritable.md), [BamReader.md](BamReader.md),
[BamWriter.md](BamWriter.md)) needs to persist an array field.

| Class | Array type | Element write/read call |
|---|---|---|
| `IoPtaDatagramFloat` (`IPD_float`) | `PTA_stdfloat`/`CPTA_stdfloat` | `Datagram::add_stdfloat()` / `DatagramIterator::get_stdfloat()` |
| `IoPtaDatagramInt` (`IPD_int`) | `PTA_int`/`CPTA_int` | `Datagram::add_uint32()` / `DatagramIterator::get_uint32()` — note: `int` elements are written/read as `uint32`, not a signed encoding |
| `IoPtaDatagramShort` (`IPD_ushort`) | `PTA_ushort`/`CPTA_ushort` (uses `pta_ushort.h` above) | (Follows the identical `add_uint32` count-prefix + per-element pattern as the other two — read the `.cxx` for the exact element call if precision matters) |

### Behavior notes

- **Format is always `[uint32 element count][elements...]`**, with no type
  tag or versioning — the reader must already know it's reading a
  `PTA_stdfloat` vs. `PTA_int` etc.; this is why there's a separate class
  per element type rather than one generic templated writer.
- **The `BamWriter*`/`BamReader*` manager parameter is accepted but unused**
  in all three `.cxx` files (`write_datagram(BamWriter *, ...)` — parameter
  left nameless) — present only so the call signature matches the
  conventional `write_datagram(manager, dest, ...)` shape other Bam I/O
  helpers use, not because these particular functions need pointer-fixup
  or object-graph services from the manager.
- **`IoPtaDatagramInt` writes `int` elements via `add_uint32()`** (an
  unsigned call) — relies on the same bit pattern round-tripping correctly
  through unsigned encode/decode, not a signed-aware format.

### Usage

```cpp
// inside some TypedWritable's write_datagram():
IoPtaDatagramFloat::write_datagram(manager, dest, get_my_float_array());

// inside the matching fillin():
_my_float_array = IoPtaDatagramFloat::read_datagram(manager, scan);
```

## See also

[TypedWritable.md](TypedWritable.md) · [BamReader.md](BamReader.md) ·
[BamWriter.md](BamWriter.md) · [README.md](README.md)
