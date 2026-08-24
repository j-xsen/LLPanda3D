# Randomizer / Mersenne

**Source:** `panda/src/mathutil/randomizer.h` (+ `.I`) · `mersenne.h` (+ `.cxx`)
**Inherits:** none (standalone value types)
**Inherited by:** (none)

`Randomizer` is the engine's general-purpose "give me a random number" class
— used by [Triangulator.md](Triangulator.md)'s retry logic and
[PerlinNoise.md](PerlinNoise.md)'s table shuffling, among others.
**`Randomizer` is a thin convenience wrapper around `Mersenne`, not an
independent algorithm** — every `Randomizer` owns exactly one `Mersenne
_mersenne` member and every method is implemented directly in terms of
`_mersenne.get_uint31()`.

`Mersenne` itself is a self-contained, line-for-line port of the classic
MT19937 Mersenne Twister reference C implementation (Nishimura & Matsumoto,
1997-2002; the BSD-style license header for the original algorithm is
preserved verbatim at the top of `mersenne.h`) — it is not a Panda-original
algorithm, just wrapped in a small C++ class interface.

## Behavior notes

- **`Mersenne` only exposes 31-bit output (`get_uint31()`, capped at
  `max_value = 0x7fffffff`)**, not the raw 32-bit MT19937 output — the top
  bit is discarded/unused, presumably to keep the result representable as a
  non-negative `long` uniformly across platforms.
- **`Randomizer::random_real(range)` maps the 31-bit integer to `[0, range)`
  by straight linear scaling**: `(range * get_uint31()) / 2^31` — not
  rejection sampling, so as with most such scaled-integer RNG wrappers the
  distribution has a very slight bias for `range` values that don't evenly
  divide `2^31` (immaterial for typical game-logic use, but worth knowing
  if used for anything statistically sensitive).
- **`random_int(range)` is `floor(random_real(range))`** — implemented in
  terms of the real-valued version, not a separate integer algorithm.
- **`random_real_unit()` returns `[-0.5, 0.5)`, not `[0, 1)`** — despite the
  name sounding like a generic unit interval, it's specifically centered on
  zero (`random_real(1.0) - 0.5`), useful for zero-mean jitter/noise without
  the caller needing to re-center it themselves.
- **Seed convention: passing `0` means "pick a random seed," any nonzero
  value is used exactly as given** — this is the same convention
  [PerlinNoise.md](PerlinNoise.md) inherits by passing its own `seed`
  parameter straight through to its internal `Randomizer`.
- **The "random" seed path (seed `0`) doesn't reseed from the OS clock every
  time — it uses one lazily-initialized static `Mersenne _next_seed`,
  seeded once from `time(nullptr)` on the very first zero-seeded
  `Randomizer` constructed process-wide, and every subsequent zero-seeded
  `Randomizer` draws its seed from that same running generator**
  (`get_next_seed()`, guarded by a static `_got_first_seed` bool). This
  means: (a) constructing many `Randomizer`s with seed 0 in a tight loop
  gives each one a genuinely different (Mersenne-derived) seed rather than
  all sharing the same clock-second-granularity time, and (b) the very
  first zero-seeded `Randomizer` in a process run is the only one whose
  ultimate seed traces back to wall-clock time at all — everything after
  it descends from the Mersenne sequence, not the clock.
- **`get_seed()` (instance method) does *not* return the seed the
  `Randomizer` was constructed with** — it draws and returns the *next*
  value from its own generator (`_mersenne.get_uint31()`), documented as "a
  unique seed value based on the seed value passed to this object (and its
  current state)." This is specifically designed for chaining — e.g.
  [PerlinNoise.md](PerlinNoise.md)'s `StackedPerlinNoise2` constructor uses
  each layer's `get_seed()` as the *next* layer's seed — and calling it
  also advances `_mersenne`'s internal state as a side effect, same as any
  other draw.
- **Copying a `Randomizer` (copy constructor / `operator=`) copies the full
  `Mersenne` state**, so a copy continues the exact same sequence from
  where the original was, not a freshly (re-)seeded generator.

## API

### Randomizer
| Signature | Notes |
|---|---|
| `explicit Randomizer(unsigned long seed = 0)` | `0` = randomly seeded (see notes); nonzero = deterministic |
| `int random_int(int range)` | `[0, range)`, via `floor(random_real(range))` |
| `double random_real(double range)` | `[0, range)`, linear scaling — see notes re: bias |
| `double random_real_unit()` | `[-0.5, 0.5)` — zero-centered, not `[0,1)` |
| `static unsigned long get_next_seed()` | The shared time-seeded generator used for `seed=0` construction |
| `unsigned long get_seed()` | Draws and returns the *next* value — see notes; not the ctor seed |

### Mersenne
| Signature | Notes |
|---|---|
| `explicit Mersenne(unsigned long seed)` | No default seed — always explicit |
| `unsigned long get_uint31()` | `[0, 0x7fffffff]` |
| `enum { max_value = 0x7fffffff }` | |

## Usage

```cpp
Randomizer rng;                    // seed = 0 -> randomly seeded
int roll = rng.random_int(6) + 1;  // 1..6
double jitter = rng.random_real_unit() * 0.1;  // +/- 0.05

Randomizer deterministic(12345);   // same sequence every run
```

## See also

[PerlinNoise.md](PerlinNoise.md) (uses `Randomizer` for table shuffling and seed chaining) ·
[Triangulator.md](Triangulator.md) (uses `Randomizer` to re-shuffle on numerical failure) ·
[README.md](README.md)
