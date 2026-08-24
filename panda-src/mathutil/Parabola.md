# LParabola

**Source:** `panda/src/mathutil/parabola_src.h/.I/.cxx` (macro-templated body) +
`parabola.h` (float/double instantiation wrapper)
**Inherits:** none (plain value type)
**Inherited by:** (none)

A parabola in parametric form `P(t) = A*t^2 + B*t + C`, where `A`, `B`, `C`
are themselves `LVecBase3`s (acceleration, initial velocity, start point) —
the natural representation for a projectile arc under constant acceleration
(gravity). Used by [collide](../collide/CollisionParabola.md)'s
`CollisionParabola` for swept-projectile collision tests, and by
[Plane.md](Plane.md)'s `intersects_parabola()`.

**Same `fltnames.h`/`dblnames.h` macro-instantiation pattern as
[Plane.md](Plane.md)/[Frustum.md](Frustum.md)** — see
[../linmath/README.md](../linmath/README.md) for the mechanism.

## Behavior notes

- **The three stored vectors are physics quantities, not geometric control
  points**: `get_a()` is acceleration (e.g. gravity), `get_b()` is initial
  velocity, `get_c()` is the start position — `calc_point(t)` is literally
  the standard kinematic position equation `p = ½at² + vt + p0` restated
  with `A` already folded to include any `½` the caller wants (the class
  itself doesn't insert a `0.5` factor — whatever `A` is passed in is
  used as-is in `A*t²`, so callers modeling gravity must pass `0.5 *
  gravity_vector` as `A` if they want standard kinematics units).
- **`xform()` transforms `A`/`B` as *vectors* (`xform_vec()`) but `C` as a
  *point* (`xform_point()`)** — correct, since acceleration and velocity
  are direction+magnitude quantities unaffected by translation, while the
  start position must translate with the rest of the scene. The header
  comment explicitly calls out that `xform_vec_general()` (used for plane
  normals) is *not* correct here.
- **`write_datagram_fixed()`/`read_datagram_fixed()` vs.
  `write_datagram()`/`read_datagram()` is the same fixed-vs-standard-width
  float distinction as elsewhere in the engine**: `_fixed` variants always
  use `add_float32()`/`add_float64()` matching the vector's actual
  `FLOATTYPE`, ignoring `Datagram::set_stdfloat_double()` — used when the
  caller wants a guaranteed-width encoding independent of the datagram's
  global float-width setting (e.g. outside a bam file); the non-`_fixed`
  variants respect that global setting via `add_stdfloat()`, which is what
  bam serialization uses.
- **The default constructor produces `A=B=C=(0,0,0)`** — a degenerate,
  "meaningless" parabola per the header comment, not an error state; no
  `is_empty()`/validity flag exists on this class at all (unlike the
  bounding-volume types).
- **No `contains`/`intersects` methods live on `LParabola` itself** — all
  intersection logic ([Plane.md](Plane.md)'s `intersects_parabola()`) is
  implemented on the *other* type, treating `LParabola` as a passive data
  holder for the quadratic coefficients.

## API

| Signature | Notes |
|---|---|
| `LParabola()` | Degenerate: A=B=C=zero |
| `LParabola(const LVecBase3 &a, const LVecBase3 &b, const LVecBase3 &c)` | Acceleration, initial velocity, start point |
| `const LVecBase3 &get_a() const` / `get_b() const` / `get_c() const` | |
| `LPoint3 calc_point(FLOATTYPE t) const` | `A*t² + B*t + C` |
| `void xform(const LMatrix4 &mat)` | A/B as vectors, C as a point — see notes |
| `void write_datagram_fixed(Datagram&) const` / `read_datagram_fixed(DatagramIterator&)` | Fixed-width float32/64, ignores stdfloat setting |
| `void write_datagram(Datagram&) const` / `read_datagram(DatagramIterator&)` | Respects `Datagram::set_stdfloat_double()`; used for bam |
| `void output(std::ostream&) const` / `write(std::ostream&, int = 0) const` | |

## Usage

```cpp
// A projectile launched from origin with velocity (5, 0, 10), under gravity:
LParabola arc(LVector3(0, 0, -9.8), LVector3(5, 0, 10), LPoint3(0, 0, 0));
LPoint3 pos_at_half_second = arc.calc_point(0.5);
```

## See also

[Plane.md](Plane.md) · [../collide/CollisionParabola.md](../collide/CollisionParabola.md) ·
[README.md](README.md)
