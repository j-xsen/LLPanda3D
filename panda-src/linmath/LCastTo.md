# lcast_to / cast_to_double / cast_to_float

**Source:** `panda/src/linmath/lcast_to.h` + `lcast_to_src.h/.I`
(`#include`d twice, once under `dbl2fltnames.h` and once under
`flt2dblnames.h` — see below) + `cast_to_double.h/.I` + `cast_to_float.h/.I`
· Library: `libp3linmath` · Notify category: `linmath`

Free functions that convert `linmath` objects between `float` and `double`
precision. `lcast_to()` is the generic, macro-driven form used internally
(via the `LCAST` macro) by templated/precision-agnostic engine code;
`cast_to_double()`/`cast_to_float()` are a fixed, explicitly-named set of
overloads meant for callers — including other, non-macro-based languages —
that can't use the `LCAST` macro.

## Behavior notes

- **`lcast_to.h` is instantiated over a *pair* of numeric types, not one** —
  it `#include`s `dbl2fltnames.h` then `lcast_to_src.h`, then
  `flt2dblnames.h` then `lcast_to_src.h` again. `dbl2fltnames.h`/
  `flt2dblnames.h` (distinct from the plain `dblnames.h`/`fltnames.h` used
  everywhere else in the module — see the [module README](README.md)) each
  define **both** `FLOATTYPE`/`FLOATNAME` (the primary type, via an internal
  `#include` of the plain name-header) **and** `FLOATTYPE2`/`FLOATNAME2` (the
  *other* type) — e.g. under `flt2dblnames.h`, `FLOATTYPE` is `float` and
  `FLOATTYPE2` is `double`. This is what lets `lcast_to_src.h` declare both
  "cast float source, produce a double result" and vice versa from the same
  `_src.h` body.
- **The same-type overloads are pure no-op passthroughs — no copy, returns a
  reference to the argument.** `lcast_to(FLOATTYPE*, const LVecBase3&)`
  (`FLOATTYPE` matching the source's own type) just `return source;` — the
  `FLOATTYPE*` first parameter exists purely to select overload resolution
  by pointer type (the pointer itself is always `nullptr`, cast from `0`),
  never dereferenced. The cross-type overloads
  (`lcast_to(FLOATTYPE2*, const LVecBase3&)`) construct and return a new
  object of the other precision by value.
- **`LCAST(numeric_type, object)` (defined in `aa_luse.h`, not
  `lcast_to.h`) is the macro application code actually calls**: `#define
  LCAST(numeric_type, object) lcast_to((numeric_type *)0, object)` — it
  supplies the dummy null pointer for overload resolution so callers just
  write `LCAST(double, some_lpoint3f)` rather than the raw
  `lcast_to((double*)0, ...)` form.
- **`lcast_to_src.h` also declares overloads converting the always-`float`
  or always-`double` `intnames.h` types** (`LVecBase2i`, `LVector3i`,
  `LPoint4i`, etc. — hardcoded as `const LVecBase2i&` etc., not templated on
  `FLOATTYPE2`, since the integer family only has one instantiation) into
  either `FLOATNAME2` result type — this is the one place `lcast_to` bridges
  the integer and floating-point families, letting e.g. an `LPoint3i` be
  cast up to `LPoint3f`/`LPoint3d`.
- **`lcast_to` covers `LVecBase*`/`LVector*`/`LPoint*`/`LQuaternion`/
  `LMatrix3`/`LMatrix4` — not [LRotation/LOrientation](LRotation.md)** (they
  aren't separately overloaded; casting one requires going through
  `LQuaternion` explicitly via construction, since `lcast_to` has no
  `LRotation`/`LOrientation`-specific overload to preserve the subclass
  type).
- **`cast_to_double()`/`cast_to_float()` are a smaller, fixed set** — just
  the `f`↔`d` direction for `LVecBase2/3/4`, `LVector2/3/4`, `LPoint2/3/4`,
  `LMatrix3`, `LMatrix4` (11 overloads each) — no `LQuaternion`, no integer
  bridging, no generic macro dispatch. Per their header comments, they exist
  "primarily for the benefit of a higher-level language that can't take
  advantage of the LCAST macro" — i.e. explicit, individually-named
  functions that a foreign-function interface can bind to directly, where
  a macro-generated overload set would be awkward to expose.

## API

| Signature | Notes |
|---|---|
| `LCAST(numeric_type, object)` (macro, `aa_luse.h`) | Preferred call form: `LCAST(double, some_float_vec)` |
| `const T &lcast_to(FLOATTYPE*, const T&)` | Same-type: no-op passthrough by reference |
| `FLOATNAME2(T) lcast_to(FLOATTYPE2*, const T&)` | Cross-type: constructs a new value in the other precision |
| `FLOATNAME2(T) lcast_to(FLOATTYPE2*, const Ti&)` | Bridges an `intnames.h` type (`LVecBase3i` etc.) up to float or double |
| `LVecBase2d/3d/4d cast_to_double(const LVecBase2f/3f/4f&)` (+ `LVector`/`LPoint` 2/3/4, `LMatrix3`, `LMatrix4`) | Fixed float→double overload set |
| `LVecBase2f/3f/4f cast_to_float(const LVecBase2d/3d/4d&)` (+ `LVector`/`LPoint` 2/3/4, `LMatrix3`, `LMatrix4`) | Fixed double→float overload set |

## Usage

```cpp
LPoint3f pf(1, 2, 3);
LPoint3d pd = LCAST(double, pf);      // via lcast_to
LPoint3d pd2 = cast_to_double(pf);    // equivalent, explicit-name form

LPoint3i pi(1, 2, 3);
LPoint3f pf2 = LCAST(float, pi);      // int -> float bridge
```

## See also

[LVecBase.md](LVecBase.md) / [LPoint.md](LPoint.md) / [LVector.md](LVector.md) /
[LMatrix.md](LMatrix.md) / [LQuaternion.md](LQuaternion.md) (the types being
converted) · [README.md](README.md) (the `fltnames.h`/`dblnames.h`/
`intnames.h` macro mechanism these build on)
