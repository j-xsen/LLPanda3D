# CoordinateSystem

**Source:** `panda/src/linmath/coordinateSystem.h/.cxx` · Library:
`libp3linmath` · Notify category: `linmath`

An enum plus a handful of free conversion/query functions describing which
of Panda's four supported 3-D coordinate conventions is in effect: which
axis is "up," and whether the system is right- or left-handed. Almost every
`CoordinateSystem cs = CS_default` parameter throughout `linmath`
([LVector::up/right/forward](LVector.md),
[LMatrix::set_rotate_mat](LMatrix.md),
[compose_matrix](ComposeMatrix.md), [LQuaternion::set_hpr](LQuaternion.md))
resolves through this same enum and `get_default_coordinate_system()`.

## Behavior notes

- **`CS_default` is not itself a coordinate system — it's a sentinel meaning
  "look up `coordinate-system` in the prc config."**
  `get_default_coordinate_system()` reads the `coordinate-system`
  `ConfigVariableEnum<CoordinateSystem>` (default value `CS_zup_right`) and
  additionally maps `CS_invalid` to `CS_zup_right` as a safety net — so this
  function never actually returns `CS_default` or `CS_invalid`, only one of
  the four real systems. Every `linmath` function taking a `CoordinateSystem
  cs = CS_default` parameter resolves it through this call internally before
  doing anything with it.
- **The four real values combine two independent binary choices**: up-axis
  (`Z` vs. `Y`) and handedness (right vs. left) — `CS_zup_right`,
  `CS_yup_right`, `CS_zup_left`, `CS_yup_left`. `X` is always "right" in
  every one of the four (see [LVector::right()](LVector.md), which is
  `(1,0,0)` unconditionally).
- **`is_right_handed(cs)` and the `IS_LEFT_HANDED_COORDSYSTEM(cs)` macro say
  the same thing in opposite polarity** — the function resolves `CS_default`
  first (and errors on `CS_invalid`/anything out of range via
  `linmath_cat.error()` + `nassertr(false, false)`); the macro is a raw
  `(cs==CS_zup_left) || (cs==CS_yup_left)` check with **no** `CS_default`
  resolution — passing `CS_default` to the macro directly gives the wrong
  answer (`false`, since `CS_default` isn't `CS_zup_left`/`CS_yup_left`).
  Code inside `linmath` itself always resolves `CS_default` via
  `get_default_coordinate_system()` before checking the macro; agent-written
  code should prefer `is_right_handed()` unless it already has a resolved
  (non-`CS_default`) value in hand.
- **String parsing accepts multiple spellings per system**
  (`parse_coordinate_system_string()`): `"default"` → `CS_default`,
  `"zup"`/`"zup-right"`/`"z-up"`/`"z-up-right"` → `CS_zup_right`, similarly
  for `"yup"` variants, and `"z-up-left"`/`"zup-left"` /
  `"y-up-left"`/`"yup-left"` for the left-handed forms — case-insensitive
  (`cmp_nocase_uh`). Any unrecognized string returns `CS_invalid`, not an
  exception.
- **`operator>>` (stream extraction) never raises `CS_invalid` up to the
  caller as an error condition** — it calls `parse_coordinate_system_string()`
  and, on failure, just logs `linmath_cat->error()` and leaves `cs` set to
  `CS_invalid`; callers reading a `CoordinateSystem` from a stream must check
  for `CS_invalid` themselves afterward.
- **`operator<<` (stream insertion) prints the enumerator name in
  `zup_right`/`yup_right`/etc. form** (underscore, not hyphen) — this is
  *not* symmetric with all the hyphenated forms `operator>>`/
  `parse_coordinate_system_string()` accept on input, only with one of them.

## API

| Signature | Notes |
|---|---|
| `enum CoordinateSystem { CS_default, CS_zup_right, CS_yup_right, CS_zup_left, CS_yup_left, CS_invalid }` | |
| `CoordinateSystem get_default_coordinate_system()` | Resolves `CS_default`/`CS_invalid` via the `coordinate-system` prc variable, default `CS_zup_right` |
| `CoordinateSystem parse_coordinate_system_string(const std::string&)` | Multiple accepted spellings per value; `CS_invalid` on no match |
| `std::string format_coordinate_system(CoordinateSystem)` | Round-trips through `operator<<` |
| `bool is_right_handed(CoordinateSystem cs = CS_default)` | Resolves `CS_default` first; errors (asserts) on `CS_invalid` |
| `#define IS_LEFT_HANDED_COORDSYSTEM(cs)` | Raw macro, no `CS_default` resolution — see behavior notes |
| `std::ostream &operator<<(std::ostream&, CoordinateSystem)` / `std::istream &operator>>(std::istream&, CoordinateSystem&)` | |

## Usage

```cpp
CoordinateSystem cs = get_default_coordinate_system();  // e.g. CS_zup_right
bool rh = is_right_handed(cs);

CoordinateSystem parsed = parse_coordinate_system_string("y-up-left");
if (parsed == CS_invalid) {
  // unrecognized string
}
```

## See also

[LVector.md](LVector.md) (`up`/`right`/`forward` axis mapping per `cs`) ·
[LMatrix.md](LMatrix.md) (`set_rotate_mat`, `convert_mat`) ·
[ComposeMatrix.md](ComposeMatrix.md) / [LQuaternion.md](LQuaternion.md)
(hpr composition uses the same axes) · [README.md](README.md)
