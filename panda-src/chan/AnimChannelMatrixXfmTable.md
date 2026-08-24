# AnimChannelMatrixXfmTable

**Source:** `panda/src/chan/animChannelMatrixXfmTable.h` / `.I` / `.cxx`
**Inherits:** [AnimChannelMatrix](AnimChannelMatrix.md)

A matrix channel driven by nine per-component sub-tables — scale (i/j/k),
shear (a/b/c), rotate (h/p/r), and translate (x/y/z) — typically loaded from
an egg file's animation data. This is the ordinary matrix channel every
skeletal joint animation actually uses; [AnimChannelMatrixDynamic](AnimChannelMatrixDynamic.md)
and [AnimChannelMatrixFixed](AnimChannelMatrixFixed.md) exist for the
procedural and constant special cases respectively.

## Behavior notes

- **Twelve tables, not nine — despite the class comment.** The header/`.cxx`
  comments say "nine sub-tables" but `compose_matrix.h`'s
  `num_matrix_components` is `12`: 3 scale + 3 shear + 3 rotate + 3
  translate, indexed by the letter string `"ijkabchprxyz"`. Shear (`a`/`b`/
  `c`) is a real, independently-settable sub-table even though it's easy to
  overlook from the class comment alone.
- **A missing sub-table isn't zero — it's a per-component default.** Any
  table left empty falls back to `get_default_value()`: `1.0` for the three
  scale components, `0.0` for shear/rotate/translate. `set_table()` can also
  be handed an empty `CPTA_stdfloat` via `clear_table()`, restoring that
  default.
- **A one-entry table is a constant, any-length table is expected to match
  the bundle's frame count.** `set_table()` allows a table of size `0`
  (cleared), `1` (constant across all frames), or exactly
  `_root->get_num_frames()`; anything else in between raises an assert
  (`nassert_raise("mismatched number of frames")`) and the table is left
  unchanged. Lookups always use `table[frame % table.size()]`, so a
  size-`1` table naturally broadcasts to every frame via the modulo.
- **`has_changed()` only compares two candidate frames, not the whole
  animation.** It checks whether `table[last_frame % size]` differs from
  `table[this_frame % size]` (and, if `last_frac != this_frac`, also
  against `table[(this_frame+1) % size]` for the upcoming blend target) —
  across all twelve tables. Tables of size `0` or `1` are skipped entirely
  since they can't vary between frames.
- **`get_value_no_scale_shear()` hardcodes unit scale and zero shear**
  rather than reading `_tables[0..5]` — components `0`-`5` are set to
  `1,1,1,0,0,0` directly in the local array before only components `6-11`
  (rotate/translate) are pulled from their tables. This is cheaper than
  calling `get_value()` and stripping components afterward.
- **`get_quat()` re-decomposes hpr from the raw tables independently of
  `get_hpr()`** — it doesn't call `get_hpr()` internally, it duplicates the
  same per-table lookup loop and then calls `quat.set_hpr(hpr)`. Functionally
  identical, just not implemented in terms of the other.
- **FFT compression (`compress-channels` Config.prc variable) is a legacy,
  deprecated bam-writing path.** `write_datagram()` warns "FFT compression
  of animations is deprecated" whenever `compress_channels` is true, and
  falls back to uncompressed if `FFTCompressor::is_compression_available()`
  is false. Reading a compressed bam file still works
  (`read-compressed-channels` gates it; if disabled, `fillin()` calls
  `clear_all_tables()` and logs rather than decoding). New code should leave
  `compress-channels` at its default (off).
- **Old vs. new HPR convention is handled transparently on load.** A `false`
  `new_hpr` flag read from an old bam file triggers `old_to_new_hpr()`
  conversion of every hpr sample during `fillin()` — callers never see raw
  files' legacy angle convention.
- **`write()`'s brief-description format lists only populated sub-tables**,
  as `<letter><size>` pairs (e.g. `h30p30r30x30y30z30` for a 30-frame
  rotate+translate-only channel), printing `(no data)` if every table is
  still at its default.

## API

### Construction
| Signature | Notes |
|---|---|
| `explicit AnimChannelMatrixXfmTable(AnimGroup *parent, const std::string &name)` | Requires a real parent, unlike [AnimChannelFixed](AnimChannelFixed.md) |

### Table access
| Signature | Notes |
|---|---|
| `static bool is_valid_id(char table_id)` | Checks membership in `"ijkabchprxyz"` |
| `void set_table(char table_id, const CPTA_stdfloat &table)` | Size must be `0`, `1`, or `get_num_frames()` |
| `CPTA_stdfloat get_table(char table_id) const` | Empty `CPTA_stdfloat` if unset or `table_id` invalid |
| `bool has_table(char table_id) const` | |
| `void clear_table(char table_id)` | Resets one sub-table to its default value |
| `void clear_all_tables()` | Resets all twelve |

### Value queries (overrides from [AnimChannelMatrix](AnimChannelMatrix.md))
| Signature | Notes |
|---|---|
| `virtual void get_value(int frame, LMatrix4 &mat)` | Full compose of all twelve components |
| `virtual void get_value_no_scale_shear(int frame, LMatrix4 &value)` | Unit scale, zero shear, real rotate/translate |
| `virtual void get_scale(int frame, LVecBase3 &scale)` | From `i`/`j`/`k` tables |
| `virtual void get_hpr(int frame, LVecBase3 &hpr)` | From `h`/`p`/`r` tables |
| `virtual void get_quat(int frame, LQuaternion &quat)` | `hpr` tables converted via `set_hpr()` |
| `virtual void get_pos(int frame, LVecBase3 &pos)` | From `x`/`y`/`z` tables |
| `virtual void get_shear(int frame, LVecBase3 &shear)` | From `a`/`b`/`c` tables |
| `virtual bool has_changed(int last_frame, double last_frac, int this_frame, double this_frac)` | See Behavior notes |

## Usage

```cpp
#include "animChannelMatrixXfmTable.h"
#include "animBundle.h"
#include "pta_stdfloat.h"

void build_table_channel(AnimGroup *parent, int num_frames) {
  PT(AnimChannelMatrixXfmTable) channel =
    new AnimChannelMatrixXfmTable(parent, "joint1");

  // A per-frame Z-translate table; must be size 0, 1, or num_frames.
  PTA_stdfloat z_table;
  for (int i = 0; i < num_frames; i++) {
    z_table.push_back((PN_stdfloat)i * 0.1f);
  }
  channel->set_table('z', z_table);

  // A constant (size-1) rotation about H for every frame.
  PTA_stdfloat h_table;
  h_table.push_back(90.0f);
  channel->set_table('h', h_table);

  LMatrix4 mat;
  channel->get_value(0, mat);       // frame 0
  channel->get_value(num_frames - 1, mat);  // last frame

  bool has_z = channel->has_table('z');
  CPTA_stdfloat z_readback = channel->get_table('z');
}
```

## See also

[AnimChannelMatrix](AnimChannelMatrix.md), [AnimChannelFixed](AnimChannelFixed.md),
[AnimChannelMatrixDynamic](AnimChannelMatrixDynamic.md) (procedural counterpart),
[AnimChannelMatrixFixed](AnimChannelMatrixFixed.md) (constant counterpart),
[AnimChannelScalarTable](AnimChannelScalarTable.md) (single-table scalar analog),
[AnimBundle](AnimBundle.md) (owns `get_num_frames()`),
[MovingPartMatrix](MovingPartMatrix.md), [README.md](README.md)
