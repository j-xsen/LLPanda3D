# AnimControlCollection

**Source:** `panda/src/chan/animControlCollection.h` / `.I` / `.cxx`
**Inherits:** none (plain value class)

A named collection of [AnimControl](AnimControl.md) pointers — the
convenience layer above raw `AnimControl` objects that lets calling code
address animations by name ("walk", "run", "attack") instead of holding onto
individual `AnimControl` pointers. Higher-level tools (`Actor` in direct/,
not covered by this reference) build on this.

## Behavior notes

- **`store_anim()` replaces by name, not by ref.** Storing a new
  `AnimControl` under a name that's already in use drops the previous
  control's reference (its ref count decrements and it may be destroyed,
  unbinding its animation) rather than erroring or keeping both. If the
  replaced control was `_last_started_control`, that pointer is cleared to
  avoid dangling.
- **`_last_started_control` tracks only the most recent `play`/`loop`/`pose`
  call**, not "the currently playing one" in general — the no-argument
  overloads of `get_frame()`/`is_playing()`/`get_num_frames()` operate on
  this single remembered control, so calling `play()` on anim A then B means
  the no-arg accessors now describe B even if A is still playing underneath.
- **`unbind_anim()` re-indexes on removal.** Removing an entry shifts every
  higher index in `_controls_by_name` down by one to stay in sync with the
  now-shorter `_controls` vector — an O(n) operation, not a swap-and-pop.
- **The named single-anim methods (`play(name)`, `loop(name, restart)`,
  etc.) all silently return `false`/no-op if the name isn't found** — no
  exception, no log message. `stop_all()` is the one exception that reports
  something back: it returns `true` only if at least one anim was actually
  playing when called.
- **`which_anim_playing()` can return multiple names.** If more than one
  stored `AnimControl` is playing simultaneously, the result is all of their
  names joined by spaces — not just the single most-recent one.

## API

### Storage
| Signature | Notes |
|---|---|
| `void store_anim(AnimControl *control, const std::string &name)` | Associates/replaces by name |
| `AnimControl *find_anim(const std::string &name) const` | `nullptr` if not found |
| `bool unbind_anim(const std::string &name)` | Removes one entry |
| `void clear_anims()` | Removes all entries |
| `int get_num_anims() const` / `AnimControl *get_anim(int n) const` / `std::string get_anim_name(int n) const` | Index-based iteration; also exposed as `get_anims()`/`get_anim_names()` sequences |

### Named single-anim playback (convenience wrappers over `AnimControl`)
| Signature | Notes |
|---|---|
| `bool play(const std::string &anim_name)` / `bool play(name, from, to)` | Returns `false` if name not found |
| `bool loop(const std::string &anim_name, bool restart)` / `bool loop(name, restart, from, to)` | |
| `bool stop(const std::string &anim_name)` | |
| `bool pose(const std::string &anim_name, double frame)` | |
| `int get_frame(const std::string &anim_name) const` | `0` if not found |
| `int get_num_frames(const std::string &anim_name) const` | `0` if not found |
| `bool is_playing(const std::string &anim_name) const` | |

### All-at-once playback
| Signature | Notes |
|---|---|
| `void play_all()` / `void play_all(from, to)` | |
| `void loop_all(bool restart)` / `void loop_all(restart, from, to)` | |
| `bool stop_all()` | Returns whether anything was actually playing |
| `void pose_all(double frame)` | |

### Last-started-control accessors / misc
| Signature | Notes |
|---|---|
| `int get_frame() const` / `int get_num_frames() const` / `bool is_playing() const` | Operate on `_last_started_control`; `0`/`false` if none started yet |
| `std::string which_anim_playing() const` | Space-joined names of every currently playing control |
| `void output(std::ostream&) const` / `void write(std::ostream&) const` | `output` is a one-line count; `write` lists every name → control |

## Usage

```cpp
NodePath actor_root = window->load_model(window->get_render(), "panda-model");
AnimControlCollection anims;

NodePath walk_np = window->load_model(actor_root, "panda-walk4");
PT(PartBundle) part = DCAST(PartBundleNode,
    actor_root.find("**/+PartBundleNode").node())->get_bundle(0);
PT(AnimBundleNode) walk_node = DCAST(AnimBundleNode,
    walk_np.find("**/+AnimBundleNode").node());
anims.store_anim(part->bind_anim(walk_node->get_bundle()), "walk");

anims.loop("walk", true);
bool playing = anims.is_playing("walk");
```

## See also

[AnimControl](AnimControl.md), [PartBundle](PartBundle.md),
[AnimBundle](AnimBundle.md)
