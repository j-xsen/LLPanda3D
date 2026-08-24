# PerlinNoise / PerlinNoise2 / PerlinNoise3 / StackedPerlinNoise2 / StackedPerlinNoise3

**Source:** `panda/src/mathutil/perlinNoise.h` (+ `.I`, `.cxx`) ·
`perlinNoise2.h/.I/.cxx` · `perlinNoise3.h/.I/.cxx` ·
`stackedPerlinNoise2.h/.I/.cxx` · `stackedPerlinNoise3.h/.I/.cxx`
**Inherits:** `PerlinNoise2 : PerlinNoise`, `PerlinNoise3 : PerlinNoise` ·
`StackedPerlinNoise2`/`StackedPerlinNoise3` are standalone (compose `PerlinNoise2`/`3`, don't inherit from them)
**Inherited by:** (none)

Classic gradient (Perlin) noise, in 2-D and 3-D flavors, loosely based on
Ken Perlin's reference implementation. `PerlinNoise` is a non-published,
protected-constructor base class collecting the shared table setup —
application code always uses `PerlinNoise2`/`PerlinNoise3` directly, never
`PerlinNoise` itself. `StackedPerlinNoise2`/`3` layer multiple noise
octaves together (fractal/turbulence-style noise).

## Behavior notes

- **`PerlinNoise`'s table is a randomly-shuffled permutation of
  `[0, table_size)`, doubled in length so lookups never need modulo
  arithmetic** — `_index` is built as `0..table_size-1` shuffled via the
  owned `Randomizer`, then the same shuffled sequence is appended a second
  time (`_index[i + table_size] = _index[i]`), so any lookup index in
  `[0, 2*table_size)` is safe without wrapping.
- **`table_size` must be a power of two, and this is only checked in debug
  builds.** The constructor computes `_table_size_mask = table_size - 1`
  and verifies `(table_size ^ mask) == (table_size + mask)` (a power-of-two
  test) only under `#ifndef NDEBUG`; on failure it resets both to 0 rather
  than asserting fatally (`nassertd`'s soft-fail block) — a release build
  with a non-power-of-two `table_size` silently gets undefined/wrong
  indexing behavior instead of an error.
- **Seeding is "nonzero seed = deterministic, zero seed = random"** —
  consistent with [Randomizer.md](Randomizer.md)'s own convention, since
  `PerlinNoise`'s internal `Randomizer _randomizer` is constructed with the
  caller's seed directly (0 there means "seed from the global time-based
  sequence").
- **`PerlinNoise2`/`3` bake the noise's frequency ("scale") into a 3x3/4x4
  matrix (`_input_xform`), not a per-call multiply** — `set_scale()`
  recomputes `_input_xform = LMatrixNd::scale_mat(1/sx, 1/sy, ...) *
  _unscaled_xform`, where `_unscaled_xform` is a fixed, class-specific
  transform set up once in `init_unscaled_xform()` (not shown in the
  header; establishes the base coordinate skew the gradient-hashing scheme
  needs). Every `noise()` call transforms its input point through this
  single combined matrix rather than doing a separate scale step.
- **`grad()` picks one of 8 (2-D) / a similar small fixed set (3-D) unit-ish
  gradient directions by hashing the low bits of a table lookup**, and the
  2-D version explicitly notes it scales the 4 "edge" directions by
  `1.707` specifically "to make their lengths consistent with
  `PerlinNoise3`" — i.e. the constant was chosen to keep the 2-D and 3-D
  noise functions' output magnitude ranges comparable, not derived from a
  general formula.
- **`StackedPerlinNoise2`/`3`'s constructor builds each octave by
  successively dividing scale and multiplying amplitude**: layer `i+1`'s
  scale is layer `i`'s scale divided by `scale_factor` (higher frequency if
  `scale_factor > 1`) and its amplitude is layer `i`'s amplitude times
  `amp_scale` (weaker if `amp_scale < 1`) — the classic fractal-noise
  "each octave doubles frequency, halves amplitude" pattern generalized to
  arbitrary factors (defaults `scale_factor=4.0`, `amp_scale=0.5`).
- **Each stacked layer gets a *derived* seed, not the same seed repeated**:
  after constructing layer `i`'s `PerlinNoise2`, the loop does `seed =
  noise.get_seed()` and uses that as the next layer's seed — chaining the
  Mersenne generator forward rather than reusing one seed (which would
  otherwise make every octave identical up to scale/amplitude).
- **`add_level()`/`noise()` on the stacked classes are fully general** — you
  can build a custom stack by hand via repeated `add_level(PerlinNoise2,
  amp)` calls instead of using the convenience constructor, and `noise()`
  is simply the amplitude-weighted sum over whatever levels are present.
  `clear()` empties the stack entirely (subsequent `noise()` calls return 0
  until levels are re-added).

## API

### PerlinNoise (protected base — not directly constructible outside subclasses)
| Signature | Notes |
|---|---|
| `unsigned long get_seed()` | The seed that was actually used (resolved if the ctor arg was 0) |

### PerlinNoise2 / PerlinNoise3
| Signature | Notes |
|---|---|
| `PerlinNoise2()` / `explicit PerlinNoise2(double sx, double sy, int table_size = 256, unsigned long seed = 0)` | 3-arg-fewer default; `PerlinNoise3` adds `sz` |
| `void set_scale(double scale)` / `set_scale(sx, sy[, sz])` / `set_scale(const LVecBase2f&/LVecBase2d&)` (or `LVecBase3*` for the 3-D class) | Rebuilds `_input_xform` |
| `double noise(double x, double y[, double z]) const` / `noise(const LVecBase2f&) const` (etc.) | |
| `double operator ()(...)` overloads | Same as `noise()` |

### StackedPerlinNoise2 / StackedPerlinNoise3
| Signature | Notes |
|---|---|
| `StackedPerlinNoise2()` | Empty stack |
| `explicit StackedPerlinNoise2(double sx, double sy, int num_levels = 2, double scale_factor = 4.0, double amp_scale = 0.5, int table_size = 256, unsigned long seed = 0)` | Builds `num_levels` octaves automatically |
| `void add_level(const PerlinNoise2 &level, double amp = 1.0)` | Manual layer construction |
| `void clear()` | Empties the stack |
| `double noise(double x, double y) const` / `noise(const LVecBase2f&/d&)` | Amplitude-weighted sum over all levels |

## Usage

```cpp
StackedPerlinNoise2 terrain_noise(0.05, 0.05, 4 /* octaves */);
for (int x = 0; x < width; ++x) {
  for (int y = 0; y < height; ++y) {
    double height = terrain_noise.noise((double)x, (double)y);
  }
}
```

## See also

[Randomizer.md](Randomizer.md) (both `PerlinNoise` and its own layer chaining depend on it) ·
[README.md](README.md)
