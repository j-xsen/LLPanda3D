# PTA_LMatrix3 / PTA_LMatrix4 / PTA_LVecBase2 / PTA_LVecBase3 / PTA_LVecBase4

**Source:** `panda/src/mathutil/pta_LMatrix3.h/.cxx` · `pta_LMatrix4.h/.cxx` ·
`pta_LVecBase2.h/.cxx` · `pta_LVecBase3.h/.cxx` · `pta_LVecBase4.h/.cxx`
(each paired with a `*_ext.h` for Python interop, not documented here)
**Inherits/Inherited by:** n/a — explicit-instantiation typedef headers, not
a class hierarchy

These five headers are `mathutil`'s instance of putil's ["Pattern 2:
`pta_X`"](../putil/VectorsAndPTAs.md) — one canonical, DLL-exported
`PointerToArray<T>`/`ConstPointerToArray<T>` instantiation per linmath
vector/matrix type, so no other module needs to (and shouldn't)
re-instantiate `PointerToArray<LMatrix4f>` etc. itself. See
[../putil/VectorsAndPTAs.md](../putil/VectorsAndPTAs.md) for the general
mechanism (`EXPORT_TEMPLATE_CLASS`, the `PTA_x`/`CPTA_x` naming convention);
this file covers only what's specific to the five instantiations that live
in `mathutil`.

**Unlike putil's `IoPtaDatagram{Float,Int,Short}`, there is no dedicated Bam
I/O helper class here** — these headers are pure Pattern 2 (typedef +
explicit instantiation), nothing more. Whatever `TypedWritable` subclass
owns a `PTA_LMatrix4`/`PTA_LVecBase3`/etc. field (e.g. an animation channel
table) is responsible for writing/reading it element-by-element itself in
its own `write_datagram()`/`fillin()`, typically by looping and calling each
element's own `write_datagram()`/`read_datagram()` (see
[Parabola.md](Parabola.md)'s `write_datagram_fixed()`/`write_datagram()` for
the shape that per-element call usually takes) — there's no generic
`IoPtaDatagramLMatrix4`-style wrapper in this module.

## Behavior notes

- **`LMatrix4` and `LVecBase4` PTAs wrap the `Unaligned*` variant of the
  type, not the type itself** (`PointerToArray<UnalignedLMatrix4f>`,
  `PointerToArray<UnalignedLVecBase4f>`) — both headers' comments explain
  this is "in case we are building with SSE2 and `LMatrix4f`/`LVecBase4f`
  requires strict alignment": a `PointerToArray`'s backing storage
  (`ReferenceCountedVector<T>`) can't guarantee per-element alignment the
  way a plain `T[]` array might, so the array element type is the
  relaxed-alignment `Unaligned` variant to avoid undefined behavior/crashes
  on SSE2 builds. `LMatrix3` and `LVecBase2`/`LVecBase3` PTAs do *not* do
  this (no `pta_LMatrix3.h` comment about alignment) — only the 4-wide
  types need it, since only those are wide enough to trigger SSE2's
  16-byte alignment requirement.
- **`LVecBase2`/`LVecBase3`/`LVecBase4` each instantiate three element
  types (f/d/i — float, double, int), while `LMatrix3`/`LMatrix4` only
  instantiate two (f/d, no integer matrix type exists)** — reflecting that
  `LVecBaseNi` is a real, used type (integer vectors, e.g. for texel/pixel
  coordinates) but there's no corresponding integer matrix type in the
  engine.
- **The unqualified `PTA_LMatrix4`/`PTA_LVecBase3`/etc. (no `f`/`d`/`i`
  suffix) resolve to the `f` or `d` variant based on the build's
  `STDFLOAT_DOUBLE` setting**, exactly like `LMatrix4`/`LVecBase3`
  themselves do in `linmath` — same convention, applied consistently here.
- **The `CPPPARSER`-guarded bogus typedefs (`PTAMat4`, `PTAVecBase3f`,
  etc.) exist only for `interrogate` (Panda's Python-binding generator) and
  legacy Python code** — not part of the real C++ API surface; ignore them
  when reading C++ call sites.

## API

| Header | Typedefs produced | Element type(s) |
|---|---|---|
| `pta_LMatrix3.h` | `PTA_LMatrix3f`/`d`, `CPTA_LMatrix3f`/`d`, `PTA_LMatrix3`/`CPTA_LMatrix3` (build-default) | `LMatrix3f`, `LMatrix3d` (no alignment wrapper) |
| `pta_LMatrix4.h` | `PTA_LMatrix4f`/`d`, `CPTA_LMatrix4f`/`d`, `PTA_LMatrix4`/`CPTA_LMatrix4` | `UnalignedLMatrix4f`, `UnalignedLMatrix4d` |
| `pta_LVecBase2.h` | `PTA_LVecBase2f`/`d`/`i`, `CPTA_...`, `PTA_LVecBase2`/`CPTA_LVecBase2` | `LVecBase2f`, `LVecBase2d`, `LVecBase2i` (no alignment wrapper) |
| `pta_LVecBase3.h` | `PTA_LVecBase3f`/`d`/`i`, `CPTA_...`, `PTA_LVecBase3`/`CPTA_LVecBase3` | `LVecBase3f`, `LVecBase3d`, `LVecBase3i` (no alignment wrapper) |
| `pta_LVecBase4.h` | `PTA_LVecBase4f`/`d`/`i`, `CPTA_...`, `PTA_LVecBase4`/`CPTA_LVecBase4` | `UnalignedLVecBase4f`, `UnalignedLVecBase4d`, `UnalignedLVecBase4i` |

## Usage

```cpp
// A field on some TypedWritable animation-channel-like class:
PTA_LVecBase3 _positions;   // PointerToArray<LVecBase3f> or <LVecBase3d>

// Writing it out (in that class's own write_datagram(), not provided by mathutil):
datagram.add_uint32(_positions.size());
for (const LVecBase3 &v : _positions) {
  v.write_datagram(datagram);
}
```

## See also

[../putil/VectorsAndPTAs.md](../putil/VectorsAndPTAs.md) (the general PTA/`EXPORT_TEMPLATE_CLASS` mechanism) ·
[Parabola.md](Parabola.md) (example of the fixed-vs-standard datagram write pattern) ·
[../linmath/README.md](../linmath/README.md) · [README.md](README.md)
