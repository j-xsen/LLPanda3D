# PerspectiveLens

**Source:** `panda/src/gobj/perspectiveLens.h` (+ `.I`, `.cxx`)
**Inherits:** [Lens](Lens.md)

The common camera case: a normal field-of-view perspective projection.
This is what `Camera`/`LensNode` (`pgraph` module) default to for
application viewpoints.

## Behavior notes

**`do_compute_projection_mat()`** derives depth-mapping terms `a`/`b` from
`focal_length`/`near`/`far`, with explicit limit-case handling when either
plane is infinite: an infinite far plane (`a=1, b=-2*near`) or an infinite
near plane (`a=-1, b=2*far` — valid specifically when near/far are
inverted). The resulting `canonical` matrix layout (which components hold
`fl`, `a`, `b`, and the perspective-divide `1`/`-1`) is chosen per
`CoordinateSystem`, then composed as `lens_mat_inv * canonical * film_mat`,
same pattern as the other linear lenses.

**FOV ↔ film-size conversion overrides.** `PerspectiveLens` supplies the
real trig for the base class's pure-virtual-in-spirit stubs:
`fov_to_film(fov, fl) = tan(fov/2) * fl * 2`,
`fov_to_focal_length(fov, film) = film/2 / tan(fov/2)`,
`film_to_fov(film, fl) = atan(film/2 / fl) * 2` (converted deg↔rad as
needed). These are what `Lens::do_compute_film_size()`/
`do_compute_focal_length()`/`do_compute_fov()` call into when resolving the
film-size/focal-length/FOV triad (see [Lens](Lens.md)'s behavior notes).

**Stereo projection is only real here.** If
`UF_interocular_distance` is unset, left/right matrices alias the mono
matrix (no stereo). Otherwise, the interocular offset is applied as a
translation by `±interocular_distance/2` along the coordinate system's
"left" axis, composed *before* `canonical`; if a finite
`convergence_distance` is also set, an additional convergence shift `cd` is
applied by post-multiplying `translate_mat(±cd)` onto the already-composed
left/right matrices. `cd`'s formula depends on
`stereo-lens-old-convergence` (config var, module README): the legacy
(pre-1.9, documented as incorrect) formula is
`0.25/convergence_distance * left_axis`; the corrected default formula
derives it from the actual FOV and interocular offset via `fov_to_film()`.

**`do_extrude_depth()`** overridden to the exact matrix-inverse
implementation, same as `OrthographicLens`.

## API

| Method | Notes |
|---|---|
| `PerspectiveLens()` | Default-constructs with `Lens` defaults. |
| `PerspectiveLens(hfov, vfov)` | Explicit constructor setting both FOV components immediately. |
| `is_linear()` | Always `true`. |
| `is_perspective()` | Always `true`. |
| `make_copy()` | Returns a new `PerspectiveLens` copy. |

Frustum shaping otherwise uses inherited [Lens](Lens.md) methods —
`set_fov()`/`set_focal_length()`/`set_film_size()` (any two of the triad),
`set_near_far()`, `set_interocular_distance()`/`set_convergence_distance()`
for stereo, `set_keystone()`, etc.

## Usage

```cpp
PT(PerspectiveLens) lens = new PerspectiveLens(60.0f, 45.0f);
lens->set_near_far(1.0f, 5000.0f);
```

## See also

- [Lens](Lens.md) — base class, full API and pipelined-state details
- [OrthographicLens](OrthographicLens.md) — the no-foreshortening alternative
- [MatrixLens](MatrixLens.md) — for an explicit user-supplied matrix instead
