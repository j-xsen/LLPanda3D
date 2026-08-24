# deg_2_rad / MathNumbers

**Source:** `panda/src/linmath/deg_2_rad.h/.I` + `mathNumbers.h/.I/.cxx` ·
Library: `libp3linmath` · Notify category: `linmath`

Two small, unrelated-in-class-hierarchy pieces bundled here because both are
tiny math-constant helpers: `deg_2_rad()`/`rad_2_deg()` are free conversion
functions used throughout `linmath` (`compose_matrix`,
[LMatrix](LMatrix.md) rotation constructors, [LQuaternion](LQuaternion.md))
wherever an angle in degrees needs converting to/from radians for a
`sin`/`cos`/`atan2` call; `MathNumbers` is the static-constant holder they
(and the rest of the engine) pull `pi`/`ln2` from, in `float`, `double`, and
`PN_stdfloat` flavors.

## Behavior notes

- **`deg_2_rad()`/`rad_2_deg()` are plain multiplication by a precomputed
  constant, overloaded on `float` vs. `double`** — not template functions,
  just two overloads each, so the compiler picks the precision based on the
  argument type without any cast needed at call sites.
- **`MathNumbers` provides the *same* constants in three precisions**: `_d`
  suffix (`double`), `_f` suffix (`float`), and no suffix (`PN_stdfloat` —
  Panda's build-configured "standard" float type, `float` or `double`
  depending on `STDFLOAT_DOUBLE`). All three are computed independently in
  `mathNumbers.cxx` from the same `atan`/`log` calls rather than one being
  derived from another by casting.
- **`cpi(float)`/`cpi(double)` and `cln2(float)`/`cln2(double)` are overload-
  resolution-only helpers** — the argument value is discarded; its *type* is
  what selects which precision of `pi`/`ln2` to return (used inside
  `FLOATTYPE_REPR`-style generic/templated code that has a `FLOATTYPE` value
  in hand but needs a same-precision constant). **Note the apparent
  inconsistency in `mathNumbers.cxx`: `cpi(double)`/`cln2(double)` return
  the unsuffixed `pi`/`ln2` (the `PN_stdfloat` constants), not `pi_d`/`ln2_d`**
  — on a `STDFLOAT_DOUBLE` build these are the same value, but on a default
  (single-precision `PN_stdfloat`) build, `cpi(double)` silently returns a
  `float`-precision value widened to `double`, not full double precision.
  Worth knowing if debugging unexpectedly low-precision `pi` in double-typed
  code paths.
- **All values are `static const` file-scope-initialized globals**, not
  `constexpr` — computed once via `atan(1.0)`/`log(2.0)` at static
  initialization time, not compile-time constants.

## API

### `deg_2_rad.h`
| Signature | Notes |
|---|---|
| `double deg_2_rad(double f)` / `double rad_2_deg(double f)` | Uses `MathNumbers::deg_2_rad_d`/`rad_2_deg_d` |
| `float deg_2_rad(float f)` / `float rad_2_deg(float f)` | Uses `MathNumbers::deg_2_rad_f`/`rad_2_deg_f` |

### `MathNumbers` (`mathNumbers.h`)
| Signature | Notes |
|---|---|
| `static const double pi_d / ln2_d / rad_2_deg_d / deg_2_rad_d` | Double precision |
| `static const float pi_f / ln2_f / rad_2_deg_f / deg_2_rad_f` | Single precision |
| `static const PN_stdfloat pi / ln2 / rad_2_deg / deg_2_rad` | Panda's configured standard float precision |
| `static float cpi(float) / cln2(float)` | Overload-selects `pi_f`/`ln2_f` |
| `static double cpi(double) / cln2(double)` | Overload-selects `pi`/`ln2` (not `pi_d`/`ln2_d` — see behavior notes) |

## Usage

```cpp
float angle_rad = deg_2_rad(90.0f);
double half_turn = MathNumbers::pi_d;

template <class T>
T circumference(T radius) {
  return 2 * MathNumbers::cpi(radius) * radius;  // precision matches T
}
```

## See also

[LQuaternion.md](LQuaternion.md) / [LMatrix.md](LMatrix.md) /
[ComposeMatrix.md](ComposeMatrix.md) (all consume `deg_2_rad`/`rad_2_deg`
for their angle parameters) · [README.md](README.md)
