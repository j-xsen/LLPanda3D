# FilterProperties

**Source:** `panda/src/audio/filterProperties.h` / `.I` / `.cxx`
**Inherits:** `TypedReferenceCount`

An ordered chain of DSP filter configurations, passed to
[AudioManager](AudioManager.md)`::configure_filters()` (global chain) or
[AudioSound](AudioSound.md)`::configure_filters()` (per-sound chain). A pure
value/config type — it doesn't apply anything itself, and there's no
guarantee a given backend supports what's in the chain; calling
`configure_filters()` and checking its boolean return is the only way to
find out.

## Behavior notes

- Internally a `pvector<FilterConfig>` (`ConfigVector`). Each `FilterConfig`
  is a `FilterType` tag plus 14 generic `PN_stdfloat` slots (`_a` through
  `_n`) — the private `add_filter()` helper packs whichever of these each
  `add_*` method actually uses, leaving the rest at their default `0`.
- **Append-only, order matters.** Every `add_*` method appends to the end of
  the chain via `add_filter()`; there's no way to edit or remove a single
  entry — only `clear()` the whole chain and re-add.
- **`get_config()` is public but not `PUBLISHED`** — it's intended for
  `AudioManager`/`AudioSound` backend implementations to read the chain and
  apply it, not for application code (which only ever calls the `add_*`
  builders and `clear()`).
- `add_sfxreverb()` is the widest filter, using all 14 slots and carrying 13
  defaulted parameters — the only `add_*` method callable with just the
  first argument supplied.

## API

| Signature | Notes |
|---|---|
| `void clear()` | Removes all filters |
| `void add_lowpass(cutoff_freq, resonance_q)` | |
| `void add_highpass(cutoff_freq, resonance_q)` | |
| `void add_echo(drymix, wetmix, delay, decayratio)` | |
| `void add_flange(drymix, wetmix, depth, rate)` | |
| `void add_distort(level)` | |
| `void add_normalize(fadetime, threshold, maxamp)` | |
| `void add_parameq(center_freq, bandwidth, gain)` | |
| `void add_pitchshift(pitch, fftsize, overlap)` | |
| `void add_chorus(drymix, wet1, wet2, wet3, delay, rate, depth)` | |
| `void add_sfxreverb(drylevel=0, room=-10000, roomhf=0, decaytime=1, decayhfratio=0.5, reflectionslevel=-10000, reflectionsdelay=0.02, reverblevel=0, reverbdelay=0.04, diffusion=100, density=100, hfreference=5000, roomlf=0, lfreference=250)` | Widest filter; all params defaulted |
| `void add_compress(threshold, attack, release, gainmakeup)` | |
| `const ConfigVector &get_config()` | Backend-internal; not `PUBLISHED` |

## Usage

```cpp
PT(FilterProperties) filters = new FilterProperties();
filters->add_lowpass(500.0f, 1.0f);
filters->add_echo(0.6f, 0.4f, 0.3f, 0.5f);

if (!sound->configure_filters(filters)) {
  // backend doesn't support this chain
}
```

## See also

[AudioManager](AudioManager.md), [AudioSound](AudioSound.md), [README.md](README.md)
