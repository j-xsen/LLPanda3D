# FFTCompressor

**Source:** `panda/src/mathutil/fftCompressor.h` (+ `.cxx`)
**Inherits:** none
**Inherited by:** (none)

Lossy compressor for streams of floats destined for a `Datagram` — used
by animation-channel bam serialization to shrink recorded motion curves
(position/hpr/quaternion tracks) before writing them to disk. It requires
the external **FFTW** library to actually do anything; without it (`HAVE_FFTW`
undefined at build time), every operation transparently falls back to
lossless, uncompressed output instead of failing.

Per the class comment, it "doesn't do any real compression on its own" — it
transforms the data (via FFT) and quantizes/truncates coefficients based on
`quality`, producing a stream of small integers that a general-purpose
compressor (gzip, as applied to the whole bam file) then compresses much
more effectively than it could the original raw floats.

## Behavior notes

- **`is_compression_available()` is a compile-time fact, not a runtime
  check** — it's a hardcoded `#ifdef HAVE_FFTW` `true`/`false`; there's no
  scenario where it changes during a program's lifetime.
- **`set_quality()`'s `quality` argument has several distinct meanings
  packed into one int**, and only some are documented for normal use:
  - `-1` (negative): read `_fft_offset`/`_fft_factor`/`_fft_exponent`
    directly from the `fft-offset`/`fft-factor`/`fft-exponent`
    `ConfigVariableDouble`s in `config_mathutil` rather than deriving them.
  - `0`–`100`: normal quality dial. Internally split into three tiers with
    different offset/factor interpolation ranges (`0–40`, `40–95`,
    `95–100`) — each tier linearly interpolates `_fft_offset`/`_fft_factor`
    between fixed endpoints while holding `_fft_exponent` at a constant
    `4.0` throughout all three tiers.
  - `101`+: lossless output — the automatic fallback when `HAVE_FFTW` isn't
    compiled in, forced via `set_quality(-1)` clamped internally to `101`
    whenever FFTW is unavailable (regardless of what the caller asked for).
  - `102`/`103`/`104`: **debug-only** overrides (guarded by `#ifndef
    NDEBUG` at the call sites that check them, per the header comment) —
    102 writes all 4 quaternion components instead of 3, 103 converts
    hpr to a 9-component matrix instead of a quaternion, 104 writes raw
    hpr directly. Not meant for production bam files.
- **`write_reals()`'s array-length special cases bypass FFT entirely**:
  length 0 writes nothing after the count; length 1 just writes that one
  value with `add_stdfloat()` — FFT only kicks in for length ≥ 2. Every
  call (regardless of length or quality) writes a `uint32` length prefix
  first, so the reader always knows how many values are coming.
- **The error-threshold rejection path (`set_use_error_threshold()`) is
  currently dead code** — the `.cxx` has the actual threshold check
  commented out with a note "This logic needs a closer examination. Not
  sure it's useful as-is," while `write_reals()` still unconditionally
  writes a `reject_compression` bool (always `false` in the current build)
  ahead of the compressed payload, so the wire format has a slot for
  per-run "this run was written uncompressed because it was too erratic"
  even though nothing currently sets it true.
- **`write_header()`/`read_header()` must bracket every use of
  `write_reals()`/`read_reals()`/`write_hprs()`/`read_hprs()` on a given
  datagram** — the header records `_quality` (and, only if negative, the
  three raw FFT tuning doubles) so the reader knows how to interpret what
  follows; there's no other way to recover the compression parameters from
  the compressed stream itself. `read_header()` additionally takes a
  `bam_minor_version`, since the exact wire format has evolved across bam
  versions.
- **`set_transpose_quats()` exists purely for backward compatibility with
  old bam files** that (per the doc comment) were written with quaternions
  "inadvertently transposed" — a historical bug-compatibility flag, not
  something new code should need to set.
- **`RunWidth`'s byte-per-run encoding packs width and length into one
  byte** (`RW_width_mask = 0xc0`, `RW_length_mask = 0x3f`) with widths
  `RW_0`/`RW_8`/`RW_16`/`RW_32` in the top 2 bits and up to 63 in the
  bottom 6 — except `RW_double` (`0xff`), a reserved full-byte sentinel for
  a distinct wider encoding, handled as a special case rather than fitting
  the width/length split.

## API

| Signature | Notes |
|---|---|
| `FFTCompressor()` | Default: `quality = -1` (config-driven) |
| `static bool is_compression_available()` | Compile-time `HAVE_FFTW` check |
| `void set_quality(int quality)` / `int get_quality() const` | See tiered meaning above |
| `void set_use_error_threshold(bool)` / `get_use_error_threshold() const` | Currently has no effect — see notes |
| `void set_transpose_quats(bool)` / `get_transpose_quats() const` | Bam backward-compat only |
| `void write_header(Datagram&)` / `bool read_header(DatagramIterator&, int bam_minor_version)` | Must bracket every read/write pair below |
| `void write_reals(Datagram&, const PN_stdfloat *array, int length)` | |
| `bool read_reals(DatagramIterator&, vector_stdfloat &array)` | |
| `void write_hprs(Datagram&, const LVecBase3 *array, int length)` | |
| `bool read_hprs(DatagramIterator&, pvector<LVecBase3> &array[, bool new_hpr])` | |
| `static void free_storage()` | Releases FFTW's static plan cache (`_real_compress_plans`/`_real_decompress_plans`) |

## Usage

```cpp
FFTCompressor compressor;
compressor.set_quality(60);
compressor.write_header(datagram);
compressor.write_reals(datagram, position_track, num_frames);

// elsewhere, reading back:
FFTCompressor reader;
reader.read_header(di, bam_minor_version);
vector_stdfloat positions;
reader.read_reals(di, positions);
```

## See also

[README.md](README.md)
