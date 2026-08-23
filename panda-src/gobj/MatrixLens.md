# MatrixLens

**Source:** `panda/src/gobj/matrixLens.h` (+ `.I`, `.cxx`)
**Inherits:** [Lens](Lens.md)

A completely generic linear lens that takes an explicit, user-supplied
projection matrix instead of being parameterized by FOV/focal-length/film
size. Intended for low-level code that already has a projection matrix in
hand (e.g. matching an external camera calibration or a hand-built shadow
frustum) and doesn't want to reverse-engineer it into `Lens`'s FOV-based
API.

## Behavior notes

**Default film size is 2, not 1.** The constructor calls
`set_film_size(2.0)`, giving the default film range `[-1, 1]` in both X and
Y — matching GL projection-matrix conventions — and, incidentally, making
the default film matrix the identity. Overriding `set_film_size()` after
construction is legal but changes the units the resulting projection is
expressed in relative to `user_mat`.

**`do_compute_projection_mat()`** composes `lens_mat_inv * user_mat *
film_mat` — i.e. `user_mat` is treated as sitting between the lens's own
view transform and the film transform, exactly like the linear term in
`PerspectiveLens`/`OrthographicLens`'s own projection composition. This
means `set_view_hpr()`/`set_view_mat()` etc. (inherited from `Lens`) still
apply on top of `user_mat`, and `set_film_size()`/`set_keystone()`/
`set_custom_film_mat()` still apply via `film_mat`.

**Per-eye override matrices.** `set_left_eye_mat()`/`set_right_eye_mat()`
let a stereo `DisplayRegion` use a different projection matrix for each eye
while `user_mat` (the center matrix) is still what's used for culling.
`get_left_eye_mat()`/`get_right_eye_mat()` fall back to `user_mat` if no
per-eye override is set (`_ml_flags & MF_has_left_eye`/`MF_has_right_eye`).

**Bam-write bug:** `write_datagram()` writes `_right_eye_mat` incorrectly —
when `MF_has_right_eye` is set, it calls
`_left_eye_mat.write_datagram(dg)` a second time instead of
`_right_eye_mat.write_datagram(dg)` (`matrixLens.cxx`, the
`if (_ml_flags & MF_has_right_eye)` block). A saved `.bam`/`.pz` file
containing a `MatrixLens` with a custom right-eye matrix will silently
serialize the left-eye matrix's value in its place, and reload with the
wrong right-eye projection.

## API

| Method | Notes |
|---|---|
| `MatrixLens()` | Default-constructs with identity `user_mat`, film size 2. |
| `set_user_mat()` / `get_user_mat()` | The explicit center projection matrix; X/Y map to `[-film_size/2, film_size/2]`, Z to `[-1, 1]` (near=-1, far=1), left-handed Y-up convention. |
| `set_left_eye_mat()` / `clear_left_eye_mat()` / `has_left_eye_mat()` / `get_left_eye_mat()` | Per-eye override for stereo left channel; falls back to `user_mat`. |
| `set_right_eye_mat()` / `clear_right_eye_mat()` / `has_right_eye_mat()` / `get_right_eye_mat()` | Same for the right channel. |
| `is_linear()` | Always `true`. |
| `make_copy()` | Returns a new `MatrixLens` copy. |

## Usage

```cpp
PT(MatrixLens) lens = new MatrixLens();
lens->set_user_mat(my_custom_projection_matrix);
```

## See also

- [Lens](Lens.md) — base class; all FOV/near/far/view-transform API is
  inherited and still applies on top of `user_mat`
