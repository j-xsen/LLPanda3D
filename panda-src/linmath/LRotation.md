# LRotation / LOrientation

**Source:** `panda/src/linmath/lrotation_src.h/.I/.cxx`, `lorientation_src.*`
(`f`/`d` only) · Library: `libp3linmath` · Notify category: `linmath`
**Inherits:** [`LQuaternion`](LQuaternion.md)
**Inherited by:** (none)

`LRotation` and `LOrientation` are both unit quaternions with **no new
storage or behavior** beyond [LQuaternion](LQuaternion.md) — they exist
purely to distinguish two ways a quaternion gets used, the same naming-only
split [LPoint](LPoint.md)/[LVector](LVector.md) give `LVecBase`:

- **`LRotation`** represents a *rotation to apply* — a delta, "turn this
  much." Its constructors favor axis-angle and hpr-delta inputs.
- **`LOrientation`** represents *an absolute facing* — "point this way, with
  this twist around that direction." Its distinguishing constructor takes a
  `(point_at vector, twist)` pair rather than an axis-angle pair.

Neither class overrides any [LQuaternion](LQuaternion.md) math — `init_type()`
registers each directly under `LQuaternion` in the `TypeHandle` hierarchy,
and both are otherwise interchangeable with a plain `LQuaternion` (and with
each other) at the storage level. The split exists to make intent legible in
calling code and API signatures, not to enforce anything the compiler
checks beyond the constructor set.

## Behavior notes

- **`LOrientation(const LVector3 &point_at, FLOATTYPE twist)` builds a
  quaternion directly from the half-angle formula around `point_at`**
  (`r = cos(twist/2)`, `(i,j,k) = point_at * sin(twist/2)`) — note this
  assumes `point_at` is already a unit vector; passing a non-normalized
  vector produces a non-unit quaternion (same caveat as
  `LMatrix::set_rotate_mat_normaxis()` in [LMatrix.md](LMatrix.md)).
- **`LRotation(const LVector3 &axis, FLOATTYPE angle)` and
  `LRotation(FLOATTYPE h, FLOATTYPE p, FLOATTYPE r)`** are the two
  distinguishing constructors — the latter is equivalent to
  `LQuaternion::set_hpr()` on a default-constructed quaternion, wrapped as
  direct-initialization syntax.
- **`operator*` overload sets differ slightly to keep the return type
  meaningful**: `LRotation * LRotation` returns `LRotation` (composing two
  deltas is still a delta), `LRotation * LQuaternion` returns the more
  general `LQuaternion` (demotes, same pattern as
  [LPoint/LVector](LPoint.md)'s arithmetic demotion), and
  `LOrientation * LRotation` / `LOrientation * LQuaternion` both return
  `LOrientation` (applying a rotation delta to an orientation is still an
  orientation) — there is no `LOrientation * LOrientation` overload; composing
  two absolute orientations isn't a meaningful operation the API exposes.
- **Both classes are otherwise fully constructible from a raw `LQuaternion`
  or `LVecBase4`** (`LRotation(const LQuaternion&)`, `LOrientation(const
  LQuaternion&)`, plus the `(r,i,j,k)` component constructor) — converting
  between "this is a delta" and "this is a facing" is just a same-storage
  reinterpretation the caller does explicitly by constructing one from the
  other.
- **No integer instantiation** (like [LQuaternion](LQuaternion.md), only
  `f`/`d` exist — rotations are inherently non-integer, so the module never
  runs `intnames.h` over `lrotation_src.h`/`lorientation_src.h`).

## API

Everything from [LQuaternion](LQuaternion.md) is inherited unchanged
(`xform()`, `conjugate()`, `invert_from()`, `almost_equal()`,
`is_same_direction()`, `get_hpr()`/`set_hpr()`, `extract_to_matrix()`, ...).
Only the constructors and the `operator*` return-type overloads differ.

### LRotation
| Signature | Notes |
|---|---|
| `LRotation()` | Uninitialized |
| `LRotation(const LQuaternion &c)` / `LRotation(const LVecBase4 &copy)` | Reinterpret an existing quaternion as a rotation |
| `LRotation(FLOATTYPE r, FLOATTYPE i, FLOATTYPE j, FLOATTYPE k)` | |
| `LRotation(const LVector3 &axis, FLOATTYPE angle)` | Axis-angle (degrees) |
| `LRotation(const LMatrix3 &m)` / `LRotation(const LMatrix4 &m)` | Via `set_from_matrix()` |
| `LRotation(FLOATTYPE h, FLOATTYPE p, FLOATTYPE r)` | Heading/pitch/roll |
| `operator*(FLOATTYPE) const` / `operator/(FLOATTYPE) const` | |
| `LRotation operator*(const LRotation&) const` | Stays `LRotation` |
| `LQuaternion operator*(const LQuaternion&) const` | Demotes to `LQuaternion` |

### LOrientation
| Signature | Notes |
|---|---|
| `LOrientation()` | Uninitialized |
| `LOrientation(const LQuaternion &c)` | Reinterpret an existing quaternion as an orientation |
| `LOrientation(FLOATTYPE r, FLOATTYPE i, FLOATTYPE j, FLOATTYPE k)` | |
| `LOrientation(const LVector3 &point_at, FLOATTYPE twist)` | Facing direction + twist around it — see behavior notes for the unit-vector requirement |
| `LOrientation(const LMatrix3 &m)` / `LOrientation(const LMatrix4 &m)` | Via `set_from_matrix()` |
| `LOrientation operator*(const LRotation&) const` / `operator*(const LQuaternion&) const` | Both stay `LOrientation` — applying a delta to a facing yields a facing |

## Usage

```cpp
// LRotation: a delta to apply repeatedly
LRotation spin(LVector3::up(CS_zup_right), 30);  // 30 deg/tick around up

// LOrientation: an absolute facing
LOrientation facing(LVector3::forward(CS_zup_right), 0);  // facing forward, no twist
facing = facing * spin;   // rotate the facing by the delta each tick
```

## See also

[LQuaternion.md](LQuaternion.md) (all shared behavior lives here — this
page covers only what's new) · [LPoint.md](LPoint.md) /
[LVector.md](LVector.md) (the analogous naming-only split over `LVecBase`) ·
[README.md](README.md)
