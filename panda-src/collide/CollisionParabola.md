# CollisionParabola

**Source:** `panda/src/collide/collisionParabola.h` (+ `.I`, `.cxx`)
**Inherits:** [CollisionSolid](CollisionSolid.md)

"A parabolic arc, or subset of an arc, similar to the path of a projectile or
falling object. It is finite, having a specific beginning and end, but it is
infinitely thin. Think of it as a wire bending from point t1 to point t2
along the path of a pre-defined parabola." Built on the `LParabola` math type
(`panda/src/mathutil`) — the standard way to swept-test a thrown/lobbed
projectile's whole flight path in one shot instead of stepping it frame by
frame.

## Behavior notes

- **`t1`/`t2` are the parabola's own parametric range, not world
  units** — they're the same `t` domain `LParabola::calc_point(t)` uses;
  match them to however you're already stepping the projectile's simulated
  time.
- **Has no exact analytic bounding volume** — `compute_internal_bounds()`
  approximates it by sampling, tuned by the
  `collision-parabola-bounds-threshold` / `collision-parabola-bounds-sample`
  Config.prc variables (see [README.md](README.md)'s config table). A very
  curved or long parabola may need a higher sample count for a tight bound.
- **`get_t()` on a resulting [CollisionEntry](CollisionEntry.md) is
  meaningful here** — it identifies *where along the arc* (in the same `t1`
  ..`t2` range) the hit occurred.

## API

| Signature | Notes |
|---|---|
| `CollisionParabola()` | |
| `explicit CollisionParabola(const LParabola &parabola, PN_stdfloat t1, PN_stdfloat t2)` | |
| `void set_parabola(const LParabola&)` / `const LParabola &get_parabola() const` | |
| `void set_t1(PN_stdfloat)` / `PN_stdfloat get_t1() const` | |
| `void set_t2(PN_stdfloat)` / `PN_stdfloat get_t2() const` | |

## Usage

```cpp
LParabola arc(LVector3(0, 0, -9.8), LVector3(5, 0, 10), LPoint3(0, 0, 0));
PT(CollisionParabola) throw_path = new CollisionParabola(arc, 0.0, 1.5);
```

## See also

[CollisionSolid.md](CollisionSolid.md) · [README.md](README.md)
