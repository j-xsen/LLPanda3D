# AnimPreloadTable

**Source:** `panda/src/chan/animPreloadTable.h` / `.I` / `.cxx`
**Inherits:** `CopyOnWriteObject`

A table of per-animation metadata (basename, base frame rate, frame count)
for one model, used to support asynchronous binding: an
[AnimControl](AnimControl.md) placeholder needs to know the frame count and
rate *before* the actual animation file has finished loading (so playback
math works immediately), and this table is where that data comes from. It's
normally built by an offline tool such as `egg-optchar`, not populated by
hand at runtime.

## Behavior notes

- **Lazy sort, not eager.** `add_anim()` and `add_anims_from()` push into an
  `ov_set` and just set `_needs_sort = true`; every read accessor
  (`get_basename()`, `get_base_frame_rate()`, `get_num_frames()`,
  `output()`, `write()`, `write_datagram()`) calls `consider_sort()` first,
  which re-sorts (by basename, alphabetically) only if the flag is set. Bulk
  insertion is therefore O(n log n) once at first read, not O(n log n) per
  `add_anim()` call.
- **`add_anim()` does not check for or replace duplicates** — the header
  comment on `find_anim()` explains lookup, but `add_anim()` itself has no
  dedup check in its own doc comment; a second `add_anim()` call with the
  same basename simply inserts a second entry into the underlying `ov_set`
  (an ordered vector, not a true set keyed for uniqueness here). Check
  `find_anim()` first if you need replace-not-append semantics.
- **`add_anims_from()` says "the record in this one supersedes"** in a
  duplicate-name conflict, per its doc comment — in practice this depends on
  `ov_set`'s insert/sort behavior when duplicate keys exist after the next
  `consider_sort()`, since the merge itself is a raw append with no
  duplicate-skip logic.
- **Basename convention:** `Filename::get_basename_wo_extension()` — no
  directory, no extension — is the expected format for every basename
  passed to `add_anim()`/`find_anim()`.
- **Index numbers are invalidated by mutation.** `add_anim()`,
  `remove_anim()`, and `add_anims_from()` all can reorder the underlying
  sorted vector; don't cache an index across a call that adds or removes a
  record.

## API

| Signature | Notes |
|---|---|
| `int get_num_anims() const` | |
| `int find_anim(const std::string &basename) const` | Returns index, or `-1` |
| `std::string get_basename(int n) const` | |
| `PN_stdfloat get_base_frame_rate(int n) const` | |
| `int get_num_frames(int n) const` | |
| `void add_anim(const std::string &basename, PN_stdfloat base_frame_rate, int num_frames)` | See Behavior notes re: duplicates |
| `void add_anims_from(const AnimPreloadTable *other)` | Bulk-merges another table |
| `void remove_anim(int n)` | Renumbers following indices |
| `void clear_anims()` | |
| `void output(std::ostream&) const` / `void write(std::ostream&, int indent_level) const` | |

## Usage

```cpp
PT(AnimPreloadTable) table = new AnimPreloadTable;
table->add_anim("panda-walk4", 24.0f, 60);
table->add_anim("panda-run", 24.0f, 30);

int index = table->find_anim("panda-walk4");
if (index >= 0) {
  int num_frames = table->get_num_frames(index);
}
```

## See also

[AnimControl](AnimControl.md), [BindAnimRequest](BindAnimRequest.md),
[AnimBundle](AnimBundle.md)
