# AnimChannelScalarTable

**Source:** `panda/src/chan/animChannelScalarTable.h` / `.I` / `.cxx`
**Inherits:** [AnimChannelScalar](AnimChannelScalar.md)

A scalar channel driven by a single per-frame table, typically loaded from
an egg file — the scalar counterpart to
[AnimChannelMatrixXfmTable](AnimChannelMatrixXfmTable.md), but with exactly
one table instead of twelve. Used for morph/slider animation (e.g. a
blink or viseme channel) where the table is baked at model-authoring time
rather than driven from code (see
[AnimChannelScalarDynamic](AnimChannelScalarDynamic.md) for that case).

## Behavior notes

- **A missing table means `0.0`, not an assert or exception.**
  `get_value()` returns `0.0f` outright if `_table` is empty — there's no
  equivalent of the matrix channel's per-component default value table,
  since there's only one component here.
- **A one-entry table is a constant, any-length table must match the
  bundle's frame count.** `set_table()` allows size `0` (cleared), `1`
  (broadcasts to every frame via `table[frame % 1] == table[0]`), or exactly
  `_root->get_num_frames()`; any other size raises `nassert_raise
  ("mismatched number of frames")` and leaves the existing table untouched.
- **`has_changed()` skips comparison entirely for size-`0` or size-`1`
  tables** (the `if (_table.size() > 1)` guard) — a constant or empty table
  can never register a change, avoiding pointless modulo/index work every
  frame for the common "unanimated slider" case.
- **Bam compression has a discrete-value fast path distinct from
  `AnimChannelMatrixXfmTable`'s FFT-only compression.** When
  `compress-channels` is on, `write_datagram()` first checks whether the
  table has at most 16 distinct values (rounded to the nearest 1/1000, per
  `scale = 1000.0f`) — common for blink/viseme channels that only ever hit a
  few discrete poses — and if so writes a small index table plus 4-bit
  packed indices (two samples per byte) instead of running the lossy FFT
  compressor. Only tables with more than 16 distinct values fall through to
  `FFTCompressor`. `fillin()` mirrors this: an `index_length` byte of `0xff`
  signals the continuous/FFT-compressed path, anything less signals the
  discrete index path.
- **Reading is gated by the same `read-compressed-channels` Config.prc
  variable used by the matrix table channel** — if compressed data is
  present in the bam file but this variable is off, the channel silently
  ends up with an empty table (logged via `chan_cat.info()`, not an error).
- **`write()`'s brief description is just the table's frame count**
  (`out << _table.size()`), unlike the matrix table's per-letter breakdown,
  since there's nothing to enumerate beyond the one table.

## API

| Signature | Notes |
|---|---|
| `AnimChannelScalarTable(AnimGroup *parent, const std::string &name)` | Requires a real parent, unlike [AnimChannelFixed](AnimChannelFixed.md) |
| `void set_table(const CPTA_stdfloat &table)` | Size must be `0`, `1`, or `get_num_frames()` |
| `CPTA_stdfloat get_table() const` | |
| `bool has_table() const` | `_table != nullptr` |
| `void clear_table()` | Sets `_table` to `nullptr` (empty) |
| `virtual void get_value(int frame, PN_stdfloat &value)` | `0.0f` if empty, else `table[frame % table.size()]` |
| `virtual bool has_changed(int last_frame, double last_frac, int this_frame, double this_frac)` | Always `false` for size `0`/`1` tables |

## Usage

```cpp
#include "animChannelScalarTable.h"
#include "animBundle.h"
#include "pta_stdfloat.h"

void build_blink_channel(AnimGroup *parent, int num_frames) {
  PT(AnimChannelScalarTable) channel =
    new AnimChannelScalarTable(parent, "blink");

  PTA_stdfloat blink_table;
  for (int i = 0; i < num_frames; i++) {
    // 0.0 = open, 1.0 = closed; a few discrete values, compresses well.
    blink_table.push_back((i % 30 == 0) ? 1.0f : 0.0f);
  }
  channel->set_table(blink_table);

  PN_stdfloat value;
  channel->get_value(0, value);
  channel->get_value(num_frames - 1, value);

  bool has_data = channel->has_table();
  CPTA_stdfloat readback = channel->get_table();
}
```

## See also

[AnimChannelScalar](AnimChannelScalar.md), [AnimChannelFixed](AnimChannelFixed.md),
[AnimChannelScalarDynamic](AnimChannelScalarDynamic.md) (procedural counterpart),
[AnimChannelMatrixXfmTable](AnimChannelMatrixXfmTable.md) (multi-table matrix analog),
[AnimBundle](AnimBundle.md) (owns `get_num_frames()`),
[MovingPartScalar](MovingPartScalar.md), [README.md](README.md)
