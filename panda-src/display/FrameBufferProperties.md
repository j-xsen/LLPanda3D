# FrameBufferProperties

**Source:** `panda/src/display/frameBufferProperties.h` (+ `.I`, `.cxx`)
**Inherits:** (none — standalone value class) **Inherited by:** (none)

A property bag describing a framebuffer configuration: color/red/green/blue/
alpha/depth/stencil/accum bit depths, aux bitplanes, multisample/coverage
sample counts, back-buffer count, and boolean flags (stereo, indexed vs.
rgb color, force-hardware/force-software, sRGB, float color, float depth).
Used both to *request* a configuration (passed into
`GraphicsEngine::make_output()`/`get_default()`) and to describe what a
`GraphicsStateGuardian` actually ended up providing, so the two can be
compared.

## Behavior notes

- **Same "specified" bitmask pattern as [WindowProperties](WindowProperties.md),
  split across two masks.** Integer properties (depth/color/red/green/
  blue/alpha/stencil/accum bits, aux_rgba/aux_hrgba/aux_float, multisamples,
  coverage_samples, back_buffers) live in a `_property[FBP_COUNT]` array
  with bit `i` of `_specified` marking whether `_property[i]` was set.
  Boolean properties (indexed_color, rgb_color, stereo, force_hardware,
  force_software, srgb_color, float_color, float_depth) live packed into
  `_flags`, with `_flags_specified` as the parallel "was this bit set"
  mask. Unlike `WindowProperties`, there are no public `has_X()`/`clear_X()`
  per-field accessors exposed — specified-ness is only checked internally
  (by `subsumes()`, `operator==`, `add_properties()`, `get_quality()`) and
  via the aggregate `is_any_specified()`.
- **`get_color_bits()` returns `max(explicit color_bits, red+green+blue)`**
  — whichever was specified more precisely wins; `set_rgba_bits(r,g,b,a)` is
  a convenience that sets red/green/blue/alpha *and* derives `color_bits =
  r+g+b` in one call, marking all four (five, counting color_bits) as
  specified at once.
- **`subsumes(other)`** — true if `this` requests everything `other` does,
  at least as strongly: every flag `other` has specified-and-set must also
  be specified-and-set on `this`, and every integer property must be
  `>=` other's. This is a "does A satisfy B's requirements" check, distinct
  from equality.
- **`get_quality(reqs)` is a scored fit function, not a boolean match** —
  used to rank candidate framebuffer configurations a graphics driver
  offers against a requested `FrameBufferProperties`. Starts at
  `100000000` and deducts by tier: wrong hardware/software mode
  (`-10000000`), unrequested software rendering (`-2000000`), missing
  depth/color/alpha/stencil/accum entirely (`-1000000` each), missing aux
  bitplanes/stereo/sRGB/float formats/insufficient back-buffers/no
  multisamples at all (`-100000` each), insufficient *bit depth* in a
  requested channel (`-10000`), insufficient multisamples
  (`-1000`), unrequested extra bitplanes or excess resolution
  (`-50`), high color-bit-depth when only 1 bit was requested — to avoid
  NVIDIA linear 64-bit color modes messing up gamma (`-100`) — then small
  **bonuses** for extra depth bits (`+8`/bit), extra multisamples/coverage
  samples (`+2` each), and extra color/alpha/stencil/accum bits (`+1`/bit)
  *only if that property was requested at all*. Returns `0` outright if the
  color model (indexed vs. rgb) doesn't match what was required — "a
  nonfunctioning window." This scoring logic (and its exact constants) is
  the single most important thing to know about this class if choosing
  between multiple candidate pixel formats.
- **`is_basic()`** reports true only for "rgb(a) + depth and nothing else"
  — any stencil, aux plane, multisample/coverage sample, back-buffer,
  indexed color, force-hardware/software, sRGB, or float request makes it
  non-basic. Used to fast-path the common case.
- **`set_one_bit_per_channel()`** clamps every depth/color/alpha/stencil/
  accum property down to at most `1` — the convention throughout this class
  is that a value of exactly `1` means "any nonzero amount, I don't care
  how many bits," while `>1` is a literal minimum-bits request.
- **`get_default()` caches a function-local static**, built once from
  `config_display.h` vars already tabulated in [README.md](README.md)
  (`framebuffer-hardware/software/depth/alpha/stencil/accum/multisample/
  stereo/srgb/float`, `depth-bits`, `color-bits` (1 or 3 values — 3 counts
  as three separate error otherwise), `alpha-bits`, `stencil-bits`,
  `accum-bits`, `multisamples`, `back-buffers`). The deprecated
  `framebuffer-mode` config var, if set to anything, only logs an error
  explaining it's non-functional and listing the replacement vars — it has
  no other effect. If both `force_hardware` and `force_software` end up set
  (contradictory config), both are silently cleared.
- **`get_aux_mask()`/`get_buffer_mask()`** convert this structure into
  `RenderBuffer::Type` bitmask form (`RenderBuffer` is documented as a
  subsection of [GraphicsStateGuardian.md](GraphicsStateGuardian.md)) —
  the representation the GSG actually uses internally to select which
  buffers to clear/bind.
- **`setup_color_texture(tex)`/`setup_depth_texture(tex)`** pick the
  closest matching `Texture::Format` for a `Texture` that will be used as a
  render-to-texture target for this framebuffer configuration — color uses
  a fixed table of 17 candidate formats (from 1-bit-per-channel up to
  `rgba32` float) scanned in order for the first one with enough bits in
  every channel; depth just thresholds `get_depth_bits()` into
  16/24/32-bit depth formats (or `F_depth_component32` if `float_depth` is
  set). Both return `false` (while still picking *some* format) if no
  candidate had enough bits — callers should check the return value if
  exact precision matters.
- **`verify_hardware_software(props, renderer)`** compares `this` (the
  actual capabilities) against `props` (the request) purely on the
  force_hardware/force_software flags and logs a detailed `display_cat`
  error (naming the offending `renderer` driver string) if the actual
  output can't satisfy a hard hardware-or-software requirement.

## API

### Construction / comparison

| Signature | Notes |
|---|---|
| `constexpr FrameBufferProperties() = default` | All-zero/unspecified. |
| `static const FrameBufferProperties &get_default()` | Cached, built from config vars — see behavior notes. |
| `void clear()` | Resets to all-unspecified. |
| `void set_all_specified()` | Marks every field as specified (without changing values) — used when treating "actual" properties as fully authoritative. |
| `bool operator == / != (const FrameBufferProperties &) const` | Exact field-for-field, including specified-ness. |
| `bool subsumes(const FrameBufferProperties &other) const` | "Does `this` satisfy at least what `other` requires." |
| `void add_properties(const FrameBufferProperties &other)` | Merges only `other`'s specified fields onto `this`. |
| `bool is_any_specified() const` | |
| `bool is_basic() const` | True only for plain rgb(a)+depth. |
| `void set_one_bit_per_channel()` | Clamps bit-depth properties to the `1` ("any amount") convention. |
| `void output(std::ostream &out) const` | Human-readable dump of specified, nonzero fields. |

### Per-property accessors (`get_X`/`set_X`, `MAKE_PROPERTY`-wrapped; no `has_X`)

| Property | Type | Notes |
|---|---|---|
| `depth_bits`, `color_bits`, `red_bits`, `green_bits`, `blue_bits`, `alpha_bits`, `stencil_bits`, `accum_bits` | `int` | `1` conventionally means "any"; `set_rgba_bits(r,g,b,a)` sets four+derives `color_bits` in one call. |
| `aux_rgba`, `aux_hrgba`, `aux_float` | `int` (≤4, asserted) | Count of auxiliary render-target bitplanes of each kind. |
| `multisamples`, `coverage_samples` | `int` | Coverage sampling used only if hardware supports it and it's specified. |
| `back_buffers` | `int` | `0` = single-buffered (`is_single_buffered()`). |
| `indexed_color`, `rgb_color` | `bool` | Color model; mutually meaningful (a "nonfunctioning window" per `get_quality()` if neither is set). |
| `stereo` | `bool` | Also `INLINE bool is_stereo() const` alias. |
| `force_hardware`, `force_software` | `bool` | Mutually exclusive in practice — see behavior notes. |
| `srgb_color`, `float_color`, `float_depth` | `bool` | |

### Quality / conversion / texture setup

| Signature | Notes |
|---|---|
| `int get_quality(const FrameBufferProperties &reqs) const` | Scored fit against a request; `0` = unusable. See behavior notes for the full scoring breakdown. |
| `bool verify_hardware_software(const FrameBufferProperties &props, const std::string &renderer) const` | Logs a detailed error and returns false on hardware/software mismatch. |
| `int get_aux_mask() const` / `int get_buffer_mask() const` | Convert to `RenderBuffer::Type` bitmask form. |
| `bool setup_color_texture(Texture *tex) const` | Picks closest `Texture::Format` for render-to-texture; `false` if no exact-enough match found (a fallback format is still set). |
| `bool setup_depth_texture(Texture *tex) const` | Same, for depth. |

## Usage

```cpp
#include "frameBufferProperties.h"

FrameBufferProperties fbprops = FrameBufferProperties::get_default();
fbprops.set_rgb_color(true);
fbprops.set_depth_bits(24);
fbprops.set_multisamples(4);

GraphicsOutput *win = engine->make_output(
    pipe, "window1", 0, fbprops, winprops,
    GraphicsPipe::BF_require_window);
```

## See also

- [WindowProperties.md](WindowProperties.md) — the parallel property-bag for window (not framebuffer) configuration; same "specified" bitmask idiom.
- [GraphicsStateGuardian.md](GraphicsStateGuardian.md) — `RenderBuffer` (the bitmask type `get_aux_mask()`/`get_buffer_mask()` produce) is documented there; the GSG is what actually satisfies a `FrameBufferProperties` request.
- [GraphicsPipe.md](GraphicsPipe.md), [GraphicsPipeSelection.md](GraphicsPipeSelection.md) — use `get_quality()` internally when picking among several pipe/GSG candidates for a requested framebuffer.
- [README.md](README.md) — the `config_display.h` variables (`framebuffer-*`, `*-bits`, `multisamples`, `back-buffers`) read by `get_default()`.
