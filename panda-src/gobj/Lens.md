# Lens

**Source:** `panda/src/gobj/lens.h` (+ `.I`, `.cxx`)
**Inherits:** TypedWritableReferenceCount **Inherited by:** [MatrixLens](MatrixLens.md), [OrthographicLens](OrthographicLens.md), [PerspectiveLens](PerspectiveLens.md)

Abstract base for camera projection math: converts between a 2-D point on
the "film" (range `(-1,1)` in both axes, `(0,0)` center) and a 3-D point/
vector in view space, and vice versa. A `Lens` is not a scene graph node —
`LensNode`/`Camera` (`pgraph` module, see `../pgraph/README.md` if
documented) hold one or more `Lens` objects and give them a position and
orientation in the graph. `Lens` is also reused outside cameras — e.g. a
`Spotlight` (pgraphnodes, undocumented) uses a `Lens` to define its cone.
See the module README's "Lens hierarchy" section for the one-paragraph
overview this doc expands on.

## Behavior notes

**Pipelined/cycled state.** All of a `Lens`'s parameters live in a `CData`
(`CycleData` subclass) accessed through a `PipelineCycler`, the same
double-buffering mechanism `RenderState`/`TransformState` use in `pgraph` —
every getter/setter goes through a `CDReader`/`CDWriter`. This is why the
public API is a thin `INLINE` wrapper layer (`lens.I`) around protected
`do_*` methods that take a `CData*` directly.

**Lazy recomputation via two bitmasks.** `_user_flags` (`UserFlags` enum)
records which parameters the *user* has explicitly set (film width/height,
focal length, hfov/vfov, aspect ratio, view hpr/vector/mat, IOD,
convergence, keystone, min_fov, custom film mat). `_comp_flags`
(`CompFlags` enum) records which *derived* quantities (film/lens/projection
matrices and their inverses, film size, aspect ratio, view hpr/vector,
focal length, fov) are currently up to date. A getter checks its bit in
`_comp_flags`; if clear, it calls the corresponding virtual
`do_compute_*()` to rebuild the value (casting away `const` on the `CData*`
— compute-on-demand is treated as conceptually const), then sets the bit.
Any setter clears the bits of everything that setting invalidates (e.g.
`set_near()`/`set_far()` clear only `CF_projection_mat*`; `set_film_size()`
clears `CF_mat | CF_focal_length | CF_fov`).

**The film-size / focal-length / FOV triad.** Only two of `film_size`,
`focal_length`, and `fov` can be independently specified — the third is
always derived. `_film_size_seq`/`_focal_length_seq`/`_fov_seq` (each 0/1/2)
track *recency* of explicit user assignment; `do_resequence_fov_triad()`
reorders them on every set-call and evicts whichever of the other two was
set longest ago (its `UF_*` bit is cleared), so "set A, then B, then C"
always keeps B and C as user-specified and re-derives A. `set_min_fov()` is
a fourth alternate way to drive this — it specifies whichever of hfov/vfov
is the *smaller* dimension (so widening an already-wide window widens the
FOV, matching what users expect from a "size" control) and evicts using the
same triad logic.

**Change notification and the frustum-visualization geometry.**
`set_change_event()` names an event thrown (with the `Lens` itself as the
one parameter) on every mutation, via `do_throw_change_event()` — also
increments `_last_change` (an `UpdateSeq`, see `get_last_change()`), so
callers can cheaply poll for "did anything change" without subscribing to
the event. `do_throw_change_event()` additionally maintains
`_geom_data`: if `make_geometry()` was previously called and the returned
`Geom`'s vertex data is still referenced by someone besides the `Lens`
itself (`get_ref_count() == 1` means only the `Lens` holds it → drop it),
the frustum-visualization vertex positions are recomputed in place so the
existing `Geom` updates live rather than requiring a fresh
`make_geometry()` call.

**Non-linear vs. linear lenses.** The base class's `is_linear()`/
`is_perspective()`/`is_orthographic()` all default to `false`, and the base
`do_compute_projection_mat()` bakes in a coordinate-system conversion
(assumes `CS_zup_right`) with no real projection — it's the fallback for
hypothetical non-linear lens subclasses (fisheye/cylindrical, mentioned in
comments but not present in this module). `PerspectiveLens` and
`OrthographicLens` both override `do_compute_projection_mat()` with real
linear math and `is_linear() → true`; `do_extrude_depth()`'s default
implementation is a linear near/far interpolation (correct for any lens),
while the linear lenses override it to use the (cheaper, exact)
`do_extrude_depth_with_mat()` instead.

**Stereo.** `interocular_distance`/`convergence_distance` only take effect
in `PerspectiveLens::do_compute_projection_mat()` (the base class computes
`_projection_mat_left`/`_projection_mat_right` as aliases of the mono
matrix); `get_projection_mat(StereoChannel)` selects among mono/left/right.
`stereo-lens-old-convergence` (config var, see module README) toggles a
pre-1.9 convergence-math bug for compatibility.

**Keystone and custom film matrix compose, in order, into `_film_mat`:**
base film-size/offset scale → keystone shear → `_custom_film_mat`, applied
left-to-right as matrix multiplication in `do_compute_film_mat()`.

**Bam versioning.** `min_fov`, `interocular_distance`,
`convergence_distance`, `view_hpr`/`view_vector`/`view_mat` (if explicitly
set), `keystone`, and `custom_film_mat` are only written/read for bam files
at minor version ≥ 41 — older files don't carry these fields.

## API

| Method | Notes |
|---|---|
| `extrude(point2d, near_point, far_point)` | 2-D film point → near/far 3-D points defining the ray through it (two overloads: `LPoint2` or `LPoint3` input, z ignored on the latter). |
| `extrude_depth(point2d, point3d)` | Reverses `project()`'s depth component — uses `point2d`'s z (range -1..1, near..far) to recover the exact 3-D point. |
| `extrude_vec(point2d, vec3d)` | 2-D film point → view-direction vector at that point (same for all points on a linear lens). |
| `project(point3d, point2d)` | 3-D point → 2-D film point; returns `false` if the point is behind the lens or outside the frustum. Two overloads (`LPoint2` out, or `LPoint3` out with depth in z). |
| `set_film_size(w[, h])` / `get_film_size()` | Size/shape of the virtual film plane; sets units for `focal_length`. |
| `set_film_offset(x, y)` / `get_film_offset()` | Off-axis lens shift, same units as film size. |
| `set_focal_length()` / `get_focal_length()` | Alternate way to specify FOV; meaningless for `OrthographicLens`. |
| `set_fov(hfov[, vfov])` / `get_fov()` / `get_hfov()` / `get_vfov()` | Field of view in degrees. |
| `set_min_fov()` / `get_min_fov()` | FOV of the *narrower* window dimension — widening the window widens the FOV. |
| `set_aspect_ratio()` / `get_aspect_ratio()` | Height:width ratio of the generated image. |
| `set_near()`/`set_far()`/`set_near_far()`, `get_near()`/`get_far()` | Clip plane distances. |
| `get_default_near()` / `get_default_far()` (static) | Read from `default-near`/`default-far` config vars (see module README). |
| `set_view_hpr()` / `get_view_hpr()` | Lens facing direction as Euler angles. |
| `set_view_vector(dir, up)` / `get_view_vector()` / `get_up_vector()` | Facing direction as explicit axis + up vector. |
| `get_nodal_point()` | The lens's viewpoint (translation row of the view matrix). |
| `set_view_mat()` / `get_view_mat()` / `clear_view_mat()` | Arbitrary transform replacing the HPR/vector forms; affects only the projection matrix, not lighting computations. |
| `set_interocular_distance()`/`get_interocular_distance()`, `set_convergence_distance()`/`get_convergence_distance()` | Stereo params; only meaningful on `PerspectiveLens`. |
| `set_keystone()` / `get_keystone()` / `clear_keystone()` | Projector keystone correction, defaults from `default-keystone`. |
| `set_custom_film_mat()` / `get_custom_film_mat()` / `clear_custom_film_mat()` | Arbitrary extra transform on nominal `(-1,1)` film-space points. |
| `set_frustum_from_corners(ul, ur, ll, lr, flags)` | Fits a frustum to four given corner points; `flags` is a bitwise-OR of `FC_roll`/`FC_camera_plane`/`FC_off_axis`/`FC_aspect_ratio`/`FC_shear`/`FC_keystone` controlling fit tightness vs. sanity (see the `.cxx` doc comment for full semantics of each bit). |
| `recompute_all()` | Debug-only: clears all `_comp_flags` to force full recompute. |
| `is_linear()` / `is_perspective()` / `is_orthographic()` | Virtual type-testing predicates; all `false` on the base. |
| `make_geometry()` | Builds a `Geom` (see [Geom](Geom.md)) visualizing the frustum as a wireframe hexahedron (curved/segmented via `lens-geom-segments` config var for non-linear lenses); `nullptr` if not possible. |
| `make_bounds()` | Builds a `BoundingHexahedron` from the eight frustum corners via `extrude()`; `nullptr` if not possible. |
| `get_projection_mat(channel = SC_mono)` / `get_projection_mat_inv()` | 3-D → 2-D transform (and inverse); `channel` selects `SC_mono`/`SC_left`/`SC_right`. |
| `get_film_mat()` / `get_film_mat_inv()` | Point-behind-lens ↔ point-on-film. |
| `get_lens_mat()` / `get_lens_mat_inv()` | Point-in-front-of-lens ↔ point-in-space (i.e. the lens's own transform). |
| `get_last_change()` | `UpdateSeq` incremented on every property change — cheap "did anything change" check. |
| `make_copy()` | Pure virtual; each subclass returns a copy of itself. |

`StereoChannel` enum: `SC_mono`, `SC_left`, `SC_right`, `SC_stereo` (=
`SC_left | SC_right`). `FromCorners` enum: see `set_frustum_from_corners()`
above.

## Usage

```cpp
PT(PerspectiveLens) lens = new PerspectiveLens();
lens->set_fov(60.0f);
lens->set_near_far(1.0f, 1000.0f);

LPoint3 near_p, far_p;
if (lens->extrude(LPoint2(0.0f, 0.0f), near_p, far_p)) {
  // near_p/far_p define the ray through the center of the screen.
}
```

## See also

- [MatrixLens](MatrixLens.md), [OrthographicLens](OrthographicLens.md),
  [PerspectiveLens](PerspectiveLens.md) — concrete subclasses
- [Geom](Geom.md) — returned by `make_geometry()`
- [InternalName](InternalName.md) — `get_vertex()` used internally to build
  the frustum-visualization `GeomVertexData`
- Module [README](README.md) — config vars (`default-near`, `default-fov`,
  `lens-geom-segments`, `stereo-lens-old-convergence`, …), shared `GeomEnums`
