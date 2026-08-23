# OrthographicLens

**Source:** `panda/src/gobj/orthographicLens.h` (+ `.I`, `.cxx`)
**Inherits:** [Lens](Lens.md)

A linear lens with no perspective foreshortening — parallel lines stay
parallel. Field of view and focal length have no meaning here; instead the
frustum shape is controlled entirely by `film_size` (inherited from `Lens`),
which represents a planar projection window hanging in space at a fixed
size regardless of distance.

## Behavior notes

**`do_compute_projection_mat()`** builds a `canonical` orthographic matrix
(depth-only scale/bias terms `a`/`b` derived from near/far distance) whose
exact layout depends on the active `CoordinateSystem` (`CS_zup_right`/
`CS_yup_right`/`CS_zup_left`/`CS_yup_left` each get a different component
placement/sign to land Z in `[-1, 1]`), then composes it the same way the
other linear lenses do: `lens_mat_inv * canonical * film_mat`. Unlike
`PerspectiveLens`, left/right eye projection matrices are always aliased to
the mono matrix — stereo interocular offset has no effect on an
orthographic projection (there's no vanishing point to offset around), so
`set_interocular_distance()`/`set_convergence_distance()` are silently
ignored here despite being inherited public API.

**`do_extrude_depth()`** is overridden to use
`do_extrude_depth_with_mat()` (the exact matrix-inverse-based
implementation) instead of `Lens`'s generic linear-interpolation fallback,
since it's cheap and exact for any linear lens.

An unrecognized `CoordinateSystem` value logs a `gobj_cat.error()` and
falls back to an identity `canonical` matrix rather than asserting or
crashing.

## API

| Method | Notes |
|---|---|
| `OrthographicLens()` | Default-constructs (inherits `Lens`'s defaults: film size 1×1, near/far from `default-near`/`default-far`). |
| `is_linear()` | Always `true`. |
| `is_orthographic()` | Always `true`. |
| `make_copy()` | Returns a new `OrthographicLens` copy. |

All frustum-shaping is done through inherited [Lens](Lens.md) methods —
`set_film_size()`, `set_film_offset()`, `set_near_far()`, `set_keystone()`,
etc. `set_fov()`/`set_focal_length()` are inherited but have no effect on
the actual projection (they only affect internally-tracked film-size
derivation if film size isn't explicitly set).

## Usage

```cpp
PT(OrthographicLens) lens = new OrthographicLens();
lens->set_film_size(20.0f, 20.0f);  // 20x20 world units visible
lens->set_near_far(-1000.0f, 1000.0f);
```

## See also

- [Lens](Lens.md) — base class
- [PerspectiveLens](PerspectiveLens.md) — the FOV-based alternative
