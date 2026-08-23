# AudioVolumeAttrib

**Source:** `panda/src/pgraph/audioVolumeAttrib.h` (+ `.I`, `.cxx`)
**Inherits:** [RenderAttrib](RenderAttrib.md)

Applies a scale factor to the volume of positional sounds attached below a
node in the scene graph (consumed by the audio system, not the renderer).

## Behavior notes
- `_has_volume` is computed, not stored directly: `!IS_NEARLY_EQUAL(volume,
  1.0f)` — an explicit volume of `1.0` is indistinguishable from "no
  volume set" for composition purposes.
- `is_off()` and `has_volume()` are independent: an "off" attrib (ignores
  inherited volume) can still carry its own scale to apply below it.
- `make_identity()` caches a single shared static instance
  (`_identity_attrib`) found once and reused — a micro-optimization beyond
  the normal state interning.
- `compose_impl`: if the lower (`other`) attrib is off, it wins outright
  (volume isn't off, so nothing multiplies through); otherwise volumes
  multiply (`other->_volume * _volume`).
- `invert_compose_impl`: if this attrib is off, returns `other` unchanged;
  otherwise divides (`other->_volume / _volume`, guarding divide-by-zero by
  returning `1.0` volume when `_volume == 0`).
- Bam serialization comment notes the format was changed without bumping
  the bam version, since no existing files had this attrib in them yet.

## API

| Method | Notes |
|---|---|
| `static CPT(RenderAttrib) make_identity()` | Volume 1.0, not off; cached singleton |
| `static CPT(RenderAttrib) make(PN_stdfloat volume)` | |
| `static CPT(RenderAttrib) make_off()` | Ignores inherited volume, own volume 1.0 |
| `static CPT(RenderAttrib) make_default()` | Same as `make_identity()`'s values |
| `bool is_off() const` | |
| `bool has_volume() const` | True iff volume isn't ~1.0 |
| `PN_stdfloat get_volume() const` | |
| `CPT(RenderAttrib) set_volume(PN_stdfloat volume) const` | Returns a new interned attrib |

## Usage
```cpp
node_path.set_attrib(AudioVolumeAttrib::make(0.5f));  // half volume below this node
```

## See also
[RenderAttrib](RenderAttrib.md), [pgraph README — the state pipeline](README.md#the-state-pipeline)
