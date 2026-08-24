# MathUtil — Bounding Volumes & Math Utilities

**Source:** `panda/src/mathutil/` · Library: `libp3mathutil` · Notify category: `mathutil`

`mathutil` builds on [linmath](../linmath/README.md)'s vector/matrix/quaternion
primitives to provide two distinct groups of higher-level math machinery:

1. **Bounding volumes** — the [BoundingVolume](BoundingVolume.md) family
   (spheres, boxes, hexahedra, lines, planes, and CSG-style combinators) is
   what every scene-graph node's `get_bounds()` returns for view-frustum
   culling, and what [collide](../collide/README.md)'s
   `CollisionSolid::get_bounds()` returns as a cheap pre-filter before a real
   shape-vs-shape test.
2. **Standalone math utilities** — plane/frustum/parabola value types,
   polygon triangulation, Perlin noise, a Mersenne-Twister-backed RNG,
   facing-direction rotation helpers, and a lossy float-stream compressor
   used by animation-channel bam serialization. These have no shared base;
   each is an independent tool reached for by name.

## Class map

**Bounding volumes (abstract base + concrete shapes):**
```
TypedReferenceCount
└── BoundingVolume                         (BoundingVolume.md) — abstract; empty/infinite state, double-dispatch contains/extend_by/around
    └── GeometricBoundingVolume            (BoundingVolume.md) — adds point-based ops (extend_by(point), contains(point/lineseg))
        ├── FiniteBoundingVolume           (BoundingVolume.md) — adds get_min()/get_max()/get_volume()
        │   ├── BoundingSphere             (BoundingSphere.md)
        │   ├── BoundingBox                (BoundingBox.md)      — always axis-aligned
        │   └── BoundingHexahedron         (BoundingHexahedron.md) — oriented convex box; view frustums
        ├── BoundingLine                   (BoundingLine.md)     — infinite, no thickness
        ├── BoundingPlane                  (BoundingPlane.md)    — infinite half-space
        ├── OmniBoundingVolume             (OmniBoundingVolume.md) — always everything (disables culling)
        ├── UnionBoundingVolume            (CompositeBoundingVolumes.md) — CSG union of components
        └── IntersectionBoundingVolume     (CompositeBoundingVolumes.md) — CSG intersection of components
```

**Math value types (macro-instantiated over float/double, same mechanism as [linmath](../linmath/README.md)):**
```
LPlane      : LVecBase4    (Plane.md)     — Ax+By+Cz+D=0
LFrustum                   (Frustum.md)   — camera frustum params + projection matrices
LParabola                  (Parabola.md)  — P(t) = At²+Bt+C, projectile arcs
```

**Polygon triangulation:**
```
Triangulator                (Triangulator.md) — 2-D polygon (+holes) triangulation
└── Triangulator3            (Triangulator.md) — 3-D wrapper, projects to best-fit plane
```

**Procedural noise:**
```
PerlinNoise (protected base) (PerlinNoise.md)
├── PerlinNoise2              (PerlinNoise.md)
└── PerlinNoise3               (PerlinNoise.md)
StackedPerlinNoise2 / 3 (compose PerlinNoise2/3, don't inherit)  (PerlinNoise.md)
```

**Randomness:**
```
Mersenne                    (Randomizer.md) — MT19937 port
Randomizer (wraps Mersenne)  (Randomizer.md) — general-purpose RNG convenience API
```

**Free functions:**
```
heads_up() / look_at()      (LookAt.md) — build a rotation from a facing direction
rotate_to()                  (LookAt.md) — shortest rotation taking vector a onto vector b
```

**Serialization support:**
```
FFTCompressor                (FFTCompressor.md) — lossy float-stream compression for bam
PTA_LMatrix3 / PTA_LMatrix4 / PTA_LVecBase2/3/4  (PTAs.md) — PointerToArray instantiations
EventStoreVec2 / EventStoreVec3 / EventStoreMat4 (LinmathEvents.md) — legacy typedefs, no behavior
```

Not documented here (out of scope for this C++ reference):
- **`config_mathutil.h/.cxx`, `config_mathutil.N`** — module config/init
  boilerplate (registers types; declares the `fft_offset`/`fft_factor`/
  `fft_exponent`/`fft_error_threshold`/`bounds_type` config variables used by
  [FFTCompressor.md](FFTCompressor.md) and [BoundingVolume.md](BoundingVolume.md)'s
  `BoundsType`); its notify category (`mathutil`) is noted above. `.N` is an
  interrogate/parser directive file, no runtime API.
- **`p3mathutil_composite1.cxx`, `p3mathutil_composite2.cxx`** — build-system
  unity-build wrapper files, not real source.
- **`test_mathutil.cxx`, `test_tri.cxx`** — standalone test/demo programs,
  not library code.
- **`pta_LMatrix3_ext.h`, `pta_LMatrix4_ext.h`, `pta_LVecBase2_ext.h`,
  `pta_LVecBase3_ext.h`, `pta_LVecBase4_ext.h`** — Python `Extension<T>`
  interop glue for the PTA typedefs in [PTAs.md](PTAs.md), not real C++ API
  surface.

## Core concepts

**Every `BoundingVolume` query answers in bits, not a bool, and the bits
form a strength hierarchy.** `contains()` returns an `IntersectionFlags`
mask: `IF_possible` (might overlap) is the weakest true answer, `IF_some`
(definitely overlaps) is stronger, `IF_all` (the *argument* is completely
inside `this`) is strongest and asymmetric — it says nothing about the
reverse containment. `IF_dont_understand` means the specific pair of
concrete types has no implemented test (logged once via the `mathutil`
notify category, not an error). See [BoundingVolume.md](BoundingVolume.md)
for the full enum and the double-dispatch machinery
(`extend_other()`/`around_other()`/`contains_other()`, each with a
type-specific second-level dispatch) that makes "sphere vs. box" and "box
vs. sphere" both resolve to the right concrete test without a
combinatorial explosion of overloads.

**`empty` and `infinite` are independent flag bits every `BoundingVolume`
carries, not values its real data can represent.** A zero-radius
`BoundingSphere` (one point) is not the same as an empty sphere (no
points); `is_infinite()` is a degenerate state layered on top of a normally
finite shape, not a literal `inf` stored in the shape's fields. Always
check `is_empty()`/`is_infinite()` before trusting a getter like
`get_center()`/`get_min()` — most assert against both in debug builds.

**Several math value types in this module (`LPlane`, `LFrustum`,
`LParabola`) are macro-instantiated over float/double exactly the way
[linmath](../linmath/README.md)'s own `LVecBase`/`LMatrix`/`LPoint`/`LVector`
types are** — a `*_src.h`/`.I`/`.cxx` written once against `FLOATNAME(X)`/
`FLOATTYPE` macros, then a thin wrapper header (`plane.h`, `frustum.h`,
`parabola.h`) `#include`s the `_src` files twice, once after `fltnames.h`
and once after `dblnames.h`, producing `LPlanef`/`LPlaned` (etc.) plus a
build-config-driven unsuffixed typedef. See
[../linmath/README.md](../linmath/README.md) for the full explanation of
this code-generation mechanism — it's the same pattern, just applied to
`mathutil`'s own types, and isn't re-explained per-file here.

**`Randomizer` is not its own algorithm — it's a convenience wrapper around
`Mersenne`, an MT19937 port.** Every `Randomizer` instance owns exactly one
`Mersenne`, and its `random_int()`/`random_real()`/`random_real_unit()`
methods are all thin transformations of `Mersenne::get_uint31()`. This
same `Randomizer` also backs [PerlinNoise.md](PerlinNoise.md)'s table
shuffling and [Triangulator.md](Triangulator.md)'s retry-on-failure logic —
see [Randomizer.md](Randomizer.md) for the seed-chaining convention
(`get_seed()` returns the *next* draw, not the constructor's seed) that
those two consumers depend on.

**`look_at()` and `heads_up()` solve the same problem with opposite
priorities.** Given a forward direction and an up direction that aren't
perfectly perpendicular, `look_at()` matches the forward vector exactly and
lets up drift; `heads_up()` matches up exactly and lets forward drift. Both
funnel through a single `LMatrix3`-producing core implementation regardless
of whether the caller asked for an `LMatrix3`, `LMatrix4`, or
`LQuaternion` result. `rotate_to()` is a related but distinct free
function: given two (already-normalized) vectors, it finds the
shortest-path rotation taking one onto the other via Rodrigues' formula —
see [LookAt.md](LookAt.md).

## File index

| Topic | Purpose |
|---|---|
| [BoundingVolume.md](BoundingVolume.md) | Abstract base family: `BoundingVolume`/`GeometricBoundingVolume`/`FiniteBoundingVolume`; `IntersectionFlags`, double-dispatch |
| [BoundingSphere.md](BoundingSphere.md) | Center + radius bounding sphere |
| [BoundingBox.md](BoundingBox.md) | Axis-aligned bounding box (AABB) |
| [BoundingHexahedron.md](BoundingHexahedron.md) | Oriented convex box; view-frustum bounds |
| [BoundingLine.md](BoundingLine.md) | Infinite, zero-thickness line |
| [BoundingPlane.md](BoundingPlane.md) | Infinite half-space |
| [OmniBoundingVolume.md](OmniBoundingVolume.md) | Always-everything special case (disables culling) |
| [CompositeBoundingVolumes.md](CompositeBoundingVolumes.md) | `UnionBoundingVolume` / `IntersectionBoundingVolume` — CSG combinators |
| [Plane.md](Plane.md) | `LPlane` math type: `Ax+By+Cz+D=0`, intersection tests, reflection matrix |
| [Frustum.md](Frustum.md) | `LFrustum` camera frustum params + perspective/ortho projection matrices |
| [Parabola.md](Parabola.md) | `LParabola` projectile-arc math type |
| [Triangulator.md](Triangulator.md) | `Triangulator`/`Triangulator3` polygon triangulation (2-D / 3-D-via-projection) |
| [PerlinNoise.md](PerlinNoise.md) | `PerlinNoise2`/`3` + `StackedPerlinNoise2`/`3` octave stacking |
| [Randomizer.md](Randomizer.md) | `Randomizer` (general RNG) + `Mersenne` (MT19937 port) |
| [LookAt.md](LookAt.md) | `heads_up()`/`look_at()`/`rotate_to()` free functions |
| [FFTCompressor.md](FFTCompressor.md) | Lossy float-stream compression for animation-channel bam data |
| [PTAs.md](PTAs.md) | `PTA_LMatrix3`/`4`, `PTA_LVecBase2`/`3`/`4` — `PointerToArray` instantiations |
| [LinmathEvents.md](LinmathEvents.md) | `EventStoreVec2`/`3`/`Mat4` — legacy typedefs to `putil`'s `ParamValue` types |

## Status

mathutil — done (2026-08-24). See [../../README.md](../../README.md) for the
overall index across `panda/src/*` modules.
