# CollisionLine

**Source:** `panda/src/collide/collisionLine.h` (+ `.I`, `.cxx`)
**Inherits:** [CollisionRay](CollisionRay.md)

"An infinite line, similar to a CollisionRay, except that it extends in both
directions. It is, however, directional" — same origin/direction
representation as `CollisionRay`, but geometrically unbounded on both sides
of the origin rather than only forward along `direction`.

## Behavior notes

- **Reuses `CollisionRay`'s origin/direction accessors unchanged** — only
  the intersection math and visualization differ (testing both `+direction`
  and `-direction` from the origin).
- **"Directional" despite being two-sided**: the sign of `direction` still
  matters for anything that reads direction-dependent results (e.g. a
  reported surface normal's orientation relative to it), even though the
  line geometrically extends both ways.

## API

Inherits [CollisionRay](CollisionRay.md)'s constructors and
`get_origin()`/`get_direction()`/`set_origin()`/`set_direction()` unchanged
(no `set_from_lens()` override — line and ray share that method by
inheritance).

| Signature | Notes |
|---|---|
| `CollisionLine()` | |
| `explicit CollisionLine(const LPoint3 &origin, const LVector3 &direction)` | |
| `explicit CollisionLine(PN_stdfloat ox, PN_stdfloat oy, PN_stdfloat oz, PN_stdfloat dx, PN_stdfloat dy, PN_stdfloat dz)` | |

## See also

[CollisionRay.md](CollisionRay.md) · [CollisionSolid.md](CollisionSolid.md)
· [README.md](README.md)
