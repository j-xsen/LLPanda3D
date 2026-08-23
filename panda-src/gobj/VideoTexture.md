# VideoTexture

**Source:** `panda/src/gobj/videoTexture.h` (+ `.I`, `.cxx`)
**Inherits:** [Texture](Texture.md), AnimInterface (external, animation-control mixin — play/stop/loop/frame-rate; same interface used by character animation)
**Inherited by:** concrete video decoders (e.g. `MovieTexture` in `panda/src/movies`, undocumented)

Abstract base for textures whose image content is decoded from a video
stream frame-by-frame rather than loaded once from a still image. Protected
constructors — you never instantiate `VideoTexture` directly, only a
concrete subclass like `MovieTexture`. Playback is controlled through the
inherited `AnimInterface` (`play()`/`stop()`/`loop()`/`set_frame_rate()`,
etc.), same API shape as a character's `AnimControl`.

## Behavior notes

- `get_keep_ram_image()` is hardcoded to always return `true` — a
  `VideoTexture` must always retain its RAM image (each new decoded frame
  has to live in system RAM at least momentarily before/while being
  uploaded), unlike an ordinary `Texture` which can be told to drop its RAM
  copy after GPU upload.
- Frame-accurate updates are cull-traversal-driven: `has_cull_callback()`
  returns `true` and `cull_callback()` calls `reconsider_dirty()` every
  time the texture is encountered on a `Geom` during cull. This is
  explicitly *not required* for correctness (`get_ram_image()` is also
  self-updating on demand) but moves the decode-if-needed cost earlier,
  into the cull pass rather than the draw pass, per the source comment.
- `do_has_ram_image()` is frame-count-gated: it compares
  `ClockObject::get_global_clock()->get_frame_count()` against
  `_last_frame_update` and returns `false` (forcing a re-decode) if the
  global frame count has advanced since the last update — so a
  `VideoTexture`'s "do I have valid RAM image data" answer changes every
  render frame even without external modification, unlike a normal
  `Texture`.
- `set_video_size()` (called by a subclass once the video's native
  dimensions are known) also computes and applies pad size via
  `do_set_pad_size()` — Panda pads non-power-of-2 video dimensions up
  internally the same way it pads odd-sized still images, transparent to
  UV coordinates.
- Constructor forces `_compression = CM_off` on the underlying `Texture`
  — per-frame video textures are never compressed on load (would be far
  too slow to compress every decoded frame).
- The four `do_update_frame()`/`do_reload_ram_image()`/`do_can_reload()`/
  `do_adjust_this_size()` protected virtuals are the actual extension
  points a concrete subclass overrides to plug in a real decoder;
  `do_update_frame()` is pure virtual (`=0`).

## API

| Signature | Notes |
|---|---|
| `virtual bool get_keep_ram_image() const` | Always `true`. |
| `int get_video_width/get_video_height() const` | Native video dimensions (before padding); also `MAKE_PROPERTY`. |
| `virtual bool has_cull_callback() const` | Always `true`. |
| `virtual bool cull_callback(CullTraverser*, const CullTraverserData&) const` | Triggers `reconsider_dirty()`; always returns `true` (never culls the geom itself). |
| `void set_video_size(int w, int h)` *(protected)* | Called by a subclass once video dimensions are known. |
| `void clear_current_frame()` *(protected, inline)* | Reset frame-tracking state (e.g. on seek/restart). |
| `virtual void do_update_frame(Texture::CData *cdata, int frame) = 0` *(protected)* | Subclass hook: decode the given frame into `cdata`. |

## See also

- [Texture](Texture.md) — base class; most of the public texture API
  (dimensions, format, sampler state) is inherited unchanged
- `AnimInterface`, `MovieTexture` (`panda/src/movies`, undocumented)
