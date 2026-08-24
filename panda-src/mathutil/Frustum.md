# LFrustum

**Source:** `panda/src/mathutil/frustum_src.h/.I` (macro-templated body, no `.cxx`
— every method is inline) + `frustum.h` (float/double instantiation wrapper)
**Inherits:** none (plain value type, not a `TypedObject`)
**Inherited by:** (none)

A camera view frustum described by six scalars: left/right/top/bottom clip
planes at the near plane (`_l`, `_r`, `_t`, `_b`) plus `_fnear`/`_ffar`
distances. It's a pure math helper for building projection matrices and
constructing [BoundingHexahedron](BoundingHexahedron.md) view volumes — it
has no notion of position/orientation in the world (that's a `Lens`'s job,
outside this module).

**Same `fltnames.h`/`dblnames.h` macro-instantiation pattern as
[Plane.md](Plane.md) and `linmath`'s own types** — `frustum_src.h`/`.I` are
written once against `FLOATNAME(LFrustum)`/`FLOATTYPE`, and `frustum.h`
`#include`s them twice (once per float width) to produce `LFrustumf`/
`LFrustumd`, with `LFrustum` a typedef to whichever matches
`STDFLOAT_DOUBLE`. See [../linmath/README.md](../linmath/README.md) for the
mechanism; not re-explained here.

## Behavior notes

- **All six `make_*()` setters overwrite the entire frustum state** — there
  is no incremental "adjust just the far plane" API; changing the aspect
  ratio, say, means recomputing everything via `make_perspective_hfov()`/
  `make_perspective_vfov()` again.
- **`make_perspective_hfov()` vs. `make_perspective_vfov()` differ in which
  FOV is authoritative and which is derived from `aspect`** — hfov fixes
  the horizontal half-angle and derives `_t`/`_b` by dividing by `aspect`;
  vfov fixes the vertical half-angle and derives `_r`/`_l` by multiplying
  by `aspect`. `make_perspective(xfov, yfov, ...)` instead takes both FOVs
  explicitly and ignores aspect entirely (each axis's half-extent is
  computed independently from its own FOV) — the three constructors are
  not interchangeable with the same inputs unless `aspect` exactly matches
  `xfov`/`yfov`'s implied ratio.
- **`get_perspective_projection_mat()` handles infinite near or far planes
  as an explicit limit case**, not just by feeding `inf` into the general
  formula: `cinf(_ffar)` sets `c = 1, f = -2*_fnear` (infinite far plane,
  standard "infinite far" projection trick avoiding `0/0`); `cinf(_fnear)`
  sets `c = -1, f = 2*_ffar` (infinite near plane — unusual, but supported).
  Only when neither is infinite does it fall through to the general
  `(_ffar+_fnear)/(_ffar-_fnear)` formula.
- **The projection matrix layout depends on the target `CoordinateSystem`,
  with four hand-written matrix layouts** (`CS_zup_right`, `CS_yup_right`,
  `CS_zup_left`, `CS_yup_left`) rather than one canonical matrix plus a
  basis-change multiply — this is a hand-optimized special case, not a
  generic `convert_mat()` composition, for both
  `get_perspective_projection_mat()` and `get_ortho_projection_mat()`.
  Passing an unrecognized `cs` value logs a `mathutil` error and returns
  the identity matrix rather than asserting/crashing.
- **`_fnear`/`_ffar` in the default constructor are `sqrt(2)` and `10.0`**
  (not 1 and 1000, or similar "obviously placeholder" values) — an
  arbitrary-looking but real default; don't assume a default-constructed
  `LFrustum` is degenerate.
- **`get_perspective_params()` (both overloads) is the exact inverse of
  `make_perspective_vfov()`/`make_perspective()`** — reconstructs FOV(s) and
  aspect from `_t`/`_r`/`_fnear` via `atan()`. Only meaningful if the
  frustum was actually built as a symmetric perspective frustum (`_l == -_r`,
  `_b == -_t`); an asymmetric or orthographic frustum's `get_perspective_params()`
  result would not round-trip through `make_perspective_vfov()`.

## API

### Construction
| Signature | Notes |
|---|---|
| `LFrustum()` | `_fnear = sqrt(2)`, `_ffar = 10.0`, unit `[-1,1]` square |
| `void make_ortho_2D()` / `make_ortho_2D(l, r, t, b)` | `_fnear=-1, _ffar=1` |
| `void make_ortho(fnear, ffar)` / `make_ortho(fnear, ffar, l, r, t, b)` | Behaves like `gluOrtho` |
| `void make_perspective_hfov(hfov, aspect, fnear, ffar)` | Horizontal FOV authoritative |
| `void make_perspective_vfov(yfov, aspect, fnear, ffar)` | Vertical FOV authoritative; behaves like `gluPerspective` |
| `void make_perspective(xfov, yfov, fnear, ffar)` | Both FOVs explicit, aspect ignored |

### Queries
| Signature | Notes |
|---|---|
| `void get_perspective_params(yfov, aspect, fnear, ffar) const` | |
| `void get_perspective_params(xfov, yfov, aspect, fnear, ffar) const` | |
| `LMatrix4 get_perspective_projection_mat(CoordinateSystem cs = CS_default) const` | Handles infinite near/far — see notes |
| `LMatrix4 get_ortho_projection_mat(CoordinateSystem cs = CS_default) const` | |

### Public fields
| Field | Meaning |
|---|---|
| `FLOATTYPE _l, _r, _b, _t` | Near-plane clip extents |
| `FLOATTYPE _fnear, _ffar` | Near/far plane distances |

## Usage

```cpp
LFrustum frustum;
frustum.make_perspective_hfov(90.0, 16.0/9.0, 1.0, 1000.0);
LMatrix4 proj = frustum.get_perspective_projection_mat(CS_zup_right);

// Also useful for building a view-volume bounding shape:
PT(BoundingHexahedron) frustum_bounds =
  new BoundingHexahedron(frustum, false, CS_zup_right);
```

## See also

[BoundingHexahedron.md](BoundingHexahedron.md) · [Plane.md](Plane.md) ·
[../linmath/README.md](../linmath/README.md) · [README.md](README.md)
