# PNMImage — Panda3D's in-memory 2-D image toolkit

**Source:** `panda/src/pnmimage/` · Library: `libp3pnmimage` · Notify category: `pnmimage`

`pnmimage` is Panda3D's engine-level image library: an in-memory,
pixel-editable image type ([PNMImage](PNMImage.md)) and its floating-point
counterpart ([PfmFile](PfmFile.md)), plus the file-format plugin machinery
that reads/writes them, a small drawing API, and the shared header/metadata
base every one of these builds on. It's the thing behind Panda's texture
loading, screenshot saving, and any C++/Python code that wants to poke pixels
directly (as opposed to `Texture`, which is the GPU-facing wrapper around a
loaded image).

This directory documents the public C++ API of every class in
`panda/src/pnmimage`, for use without re-reading the engine source.

## Class map

```
PNMImageHeader                          (PNMImageHeader.md)  — shared size/type/maxval base
├── PNMImage                            (PNMImage.md)         — the main pixel-editable image
├── PfmFile                             (PfmFile.md)          — floating-point counterpart
├── PNMReader                           (PNMReader.md)        — abstract streaming reader base
└── PNMWriter                           (PNMWriter.md)        — abstract streaming writer base

TypedWritable
└── PNMFileType                         (PNMFileType.md)      — abstract file-format plugin base
    └── (concrete formats: PNM, PNG, JPEG, ... — in panda/src/pnmimagetypes, not yet documented)

PNMFileTypeRegistry                     (PNMFileTypeRegistry.md) — global singleton, standalone class

ReferenceCount
└── PNMBrush                            (PNMBrush.md)         — pen/fill shape+color for PNMPainter

PNMPainter                              (PNMPainter.md)       — standalone; draws into a PNMImage using brushes
```

`PNMImageHeader::PixelSpec`, `PixelSpecCount`, and `Histogram` are small
nested types used by both `PNMImageHeader` and `PNMImage`; they're documented
as subsections inside [PNMImageHeader.md](PNMImageHeader.md) rather than as
standalone files. `PNMImage::Row`/`CRow` (the `image[y][x]` indexing proxies)
are likewise documented as a subsection inside [PNMImage.md](PNMImage.md).

## Excluded from these docs

- **`panda/src/pnmimagetypes`** — the concrete `PNMFileType` subclasses (PNM,
  PNG, JPEG, TGA, BMP, SGI, SoftImage, ...) live in a separate module and are
  tracked as their own "Not started" row in the root README; nothing in this
  directory documents them individually.
- **`pnmbitio.h/.cxx`, `ppmcmap.h/.cxx`** — internal implementation helpers
  (bit-level stream I/O, PPM colormap quantization) used by specific format
  readers/writers in `pnmimagetypes`. No `PUBLISHED` class; not part of the
  public API.
- **`config_pnmimage.h/.cxx`** — no class, just the `pnmimage` notify category
  and a handful of config variables (`pfm_force_littleendian`,
  `pfm_reverse_dimensions`, `pfm_resize_gaussian`, `pfm_resize_quick`,
  `pfm_resize_radius`) that affect [PfmFile](PfmFile.md) read/write/resize
  behavior — covered under "Core concepts" below instead of as a standalone
  doc.
- **`convert_srgb.h/.I/.cxx` (+ `_sse2.cxx`)** — no class, just free functions
  (`encode_sRGB_uchar`/`decode_sRGB_float`/etc., with an SSE2 fast path) used
  internally by `PNMImage`'s color-space conversion. Covered under "Core
  concepts" below.
- **`pnmimage_base.h`** — no class, just the core `xel`/`xelval`/`pixel`
  typedefs used by everything in this module. Covered under "Core concepts"
  below.

## Core concepts (shared across the module)

**`xel` and `xelval` are the raw pixel storage types.** `xelval` is a `gray`
(an `unsigned short` when built with `PGM_BIGGRAYS`, the default — 16-bit
channels; `unsigned char` otherwise). `xel` is a `pixel` struct of exactly
three `gray`s (`r`, `g`, `b` — no alpha field; alpha is always a parallel
`xelval*` array). These typedefs, plus the `PPM_GETR`/`PPM_GETB`/etc. macros
and the `lumin_red`/`lumin_grn`/`lumin_blu` default luminance constants, come
from `pnmimage_base.h` and underlie every raw pixel access in
[PNMImage](PNMImage.md) and [PNMImageHeader](PNMImageHeader.md).

**Two accessor domains, everywhere.** Every class that stores pixel/point
data exposes both a raw/encoded form (integer `xelval` in `[0..maxval]` for
`PNMImage`, or the direct un-normalized float in `PfmFile`) and a normalized
float form in `[0..1]` (`PNMImage` only — `PfmFile` has no such split, since
it's already float-native). The `_val` suffix marks the raw form throughout
`PNMImage`'s API. Prefer the normalized/float accessors unless you have a
specific reason to touch raw storage (performance, bit manipulation, or
matching another `xelval`-based array exactly).

**Color space affects what the float accessors mean, not the raw ones.**
`PNMImage` tags itself with a `ColorSpace` (`CS_unspecified`, `CS_linear`,
`CS_sRGB`, `CS_scRGB`, ...; defined outside this module, in `colorSpace.h`).
Raw `_val` accessors always return/take the value as literally stored — no
conversion. The float accessors (`get_xel`/`set_xel`/`get_red`/...) convert
through `to_val()`/`from_val()`, which apply sRGB encode/decode (via
`convert_srgb.h`'s lookup tables or SSE2 path) when the image's `ColorSpace`
calls for it, so two images with the same float-visible color can have
different raw `_val` bytes if their color spaces differ. See
[PNMImage.md](PNMImage.md)'s Behavior notes for the full accessor-domain
breakdown, and `set_color_space()`'s conversion cost/precision notes.

**`maxval` is the raw-domain scale, independent of channel count.**
`get_maxval()` (inherited from [PNMImageHeader](PNMImageHeader.md)) is the
integer value a fully-on channel maps to — 255 for a typical 8-bit image, up
to 65535 for a 16-bit one. Changing it (`PNMImage::set_maxval()`) rescales
every stored raw value to the new range; it does not change what the float
accessors report.

**`BLOCKING`-tagged methods** (visible in several class declarations) do
real, potentially slow CPU work synchronously — filtering, distance
transforms, quantization — as opposed to the cheap accessor methods
surrounding them. There's no async variant in this module; the tag is
documentation for callers deciding whether to run something on a worker
thread.

**PFM-specific config variables** (`config_pnmimage.h`, `pnmimage` notify
category): `pfm_resize_gaussian`/`pfm_resize_quick`/`pfm_resize_radius`
control which filter [`PfmFile::resize()`](PfmFile.md) picks automatically;
`pfm_force_littleendian`/`pfm_reverse_dimensions` affect how `.pfm` files are
read/written by the format's reader/writer (in `pnmimagetypes`, not
documented here).

## File index

| Class | Purpose |
|---|---|
| [PNMImageHeader.md](PNMImageHeader.md) | Base class: size/channels/maxval/comment/type + `PixelSpec`/`Histogram` + reader/writer factory logic |
| [PNMImage.md](PNMImage.md) | The main pixel-editable image: accessors, blending, sub-image ops, filters, histogram/quantize, arithmetic operators |
| [PfmFile.md](PfmFile.md) | Floating-point 2-D point table: height/normal/depth maps, no-data handling, distortion, resampling |
| [PNMFileType.md](PNMFileType.md) | Abstract base for a file-format plugin |
| [PNMFileTypeRegistry.md](PNMFileTypeRegistry.md) | Global registry of every known `PNMFileType`, keyed by extension/magic number |
| [PNMReader.md](PNMReader.md) | Abstract streaming reader base (obtained via `make_reader()`) |
| [PNMWriter.md](PNMWriter.md) | Abstract streaming writer base (obtained via `make_writer()`) |
| [PNMBrush.md](PNMBrush.md) | Pen/fill shape+color descriptor, used by `PNMPainter` |
| [PNMPainter.md](PNMPainter.md) | Draws lines/rectangles into a `PNMImage` using brushes |

## Status

pnmimage — done (2026-08-28). Other `panda/src/*` subsystems not yet
documented — see `../../README.md` for the overall index.
