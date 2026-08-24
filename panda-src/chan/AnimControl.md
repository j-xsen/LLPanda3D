# AnimControl

**Source:** `panda/src/chan/animControl.h` / `.I` / `.cxx`
**Inherits:** `TypedReferenceCount`, `AnimInterface`, `Namable`

Controls the timing of one character/bundle animation binding — the object on
which `play()`/`loop()`/`stop()`/`pose()` are actually called. One `AnimControl`
is created per `PartBundle` × `AnimBundle` pairing (typically via
`PartBundle::bind_anim()`); it doesn't own the animation data itself, just
tracks playback state (frame, rate, looping) via the inherited
`AnimInterface` methods (see below) and links the bundle back to the
specific channel index it was bound on.

## AnimInterface (inherited play/loop/stop/pose)

`AnimControl` doesn't declare its own play/stop/loop methods — they come from
`AnimInterface` (`panda/src/putil/animInterface.h`), the base for anything
with frame-based playback:

| Signature | Notes |
|---|---|
| `void play()` / `void play(double from, double to)` | Plays once, optionally a sub-range |
| `void loop(bool restart)` / `void loop(bool restart, double from, double to)` | Loops forever; `restart` controls whether to restart from the beginning if already playing |
| `void pingpong(bool restart)` / `void pingpong(bool restart, double from, double to)` | Loops back and forth |
| `void stop()` | |
| `void pose(double frame)` | Sets a single fixed frame, no playback |
| `void set_play_rate(double) / get_play_rate() const` | Multiplier on the base frame rate |
| `double get_frame_rate() const` | Effective playing rate (`base_frame_rate * play_rate`) |
| `int get_frame() const` / `get_next_frame() const` / `double get_frac() const` | Current integer frame, the next frame for interpolation, and the fractional position between them |
| `bool is_playing() const` | |

Every play/loop/pose call above routes through the protected
`animation_activated()` callback, which `AnimControl` overrides to call
`PartBundle::control_activated(this)` — this is how a `PartBundle` knows
which `AnimControl` last started playing among possibly several bound to it.

## Behavior notes

- **Two ways to construct: eager or pending.** The public constructor always
  starts `_pending = true` with `_anim = nullptr` — it's meant to be a
  placeholder returned immediately from an async bind
  ([BindAnimRequest](BindAnimRequest.md)) before the animation has actually
  loaded. `setup_anim()` (called once, from the loading thread) supplies the
  real `AnimBundle`, channel index, and bound-joints mask and flips
  `_pending` to `false`; `fail_anim()` flips it to `false` without ever
  supplying an anim if the load failed. Both are guarded by `_pending_lock`
  and signal `_pending_cvar` so `wait_pending()` can block a caller until
  either resolves.
- **`is_pending()` vs `has_anim()`.** While pending, the `AnimControl`'s
  interface (`play()`, `get_frame()`, etc.) is fully usable and harmless —
  nothing visible happens yet. After the bind resolves, check `has_anim()`:
  `true` means it bound successfully, `false` means `fail_anim()` was called
  (bad file, missing joints, etc.) and the object is a permanent no-op stub.
- **`set_pending_done_event()` fires immediately if already resolved.** If
  the bind already finished (success or failure) by the time the event name
  is set, `throw_event()` fires right away rather than waiting for a
  bind that has already happened.
- **`get_bound_joints()` is `BitArray::all_on()` for a normal full-body
  animation**, and a strict subset only when this control resulted from a
  partial/subset bind (see [PartSubset](PartSubset.md)) — bits correspond to
  joints/sliders in depth-first LIFO bind order.
- **`channel_has_changed()` / `mark_channels()`** are the frame-skip
  optimization used internally by `PartBundle`'s update pass: `mark_channels()`
  records the current frame (and fractional frame, if frame-blending is on)
  as a checkpoint, and a later `channel_has_changed()` call asks a specific
  channel whether its value would differ between the checkpoint and now —
  letting unbound/unchanged joints skip recomputation.
- **Destructor calls back into the part.** `~AnimControl()` calls
  `get_part()->control_removed(this)` — a `PartBundle` always finds out when
  one of its controls goes away, even if the last reference was dropped by
  application code and not the bundle itself.
- **`get_part()` returns a `PartBundle*` from a `PartGroup*` field.** The
  header stores `_part` as `PT(PartGroup)` rather than `PT(PartBundle)`
  purely to avoid a circular include (`partBundle.h` isn't includable from
  `animControl.h`); it's downcast with `DCAST` in `get_part()`. It's always
  actually a `PartBundle`.

## API

| Signature | Notes |
|---|---|
| `AnimControl(const std::string &name, PartBundle *part, double frame_rate, int num_frames)` | Constructs a pending placeholder |
| `void setup_anim(PartBundle *part, AnimBundle *anim, int channel_index, const BitArray &bound_joints)` | Resolves a pending bind with real data; call once |
| `void fail_anim(PartBundle *part)` | Resolves a pending bind as a permanent failure |
| `bool is_pending() const` | Still waiting on an async load? |
| `void wait_pending()` | Blocks the calling thread until resolved |
| `bool has_anim() const` | Bound successfully? (meaningless while `is_pending()`) |
| `void set_pending_done_event(const std::string&) / get_pending_done_event() const` | Event thrown when the async bind resolves |
| `PartBundle *get_part() const` | The bundle this control is bound to |
| `AnimBundle *get_anim() const` | The bound animation, or `nullptr` while pending/failed |
| `int get_channel_index() const` | Which per-joint channel slot this control occupies |
| `const BitArray &get_bound_joints() const` | Subset mask, see Behavior notes |
| `void set_anim_model(PandaNode*) / get_anim_model() const` | Optional back-reference to the model root that owns this anim, kept alive purely by refcount |
| `void output(std::ostream&) const` | |

## Usage

```cpp
NodePath actor_root = window->load_model(window->get_render(), "panda-model");
NodePath anim_np = window->load_model(actor_root, "panda-walk4");

PT(PartBundle) part = DCAST(PartBundleNode,
    actor_root.find("**/+PartBundleNode").node())->get_bundle(0);
PT(AnimBundleNode) anim_node = DCAST(AnimBundleNode,
    anim_np.find("**/+AnimBundleNode").node());

PT(AnimControl) control = part->bind_anim(anim_node->get_bundle());
if (control != nullptr) {
  control->loop(true);          // AnimInterface method
  int current_frame = control->get_frame();
}
```

## See also

[PartBundle](PartBundle.md), [AnimBundle](AnimBundle.md),
[AnimControlCollection](AnimControlCollection.md),
[BindAnimRequest](BindAnimRequest.md), [PartSubset](PartSubset.md)
