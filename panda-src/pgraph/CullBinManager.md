# CullBinManager

**Source:** `panda/src/pgraph/cullBinManager.h` (+ `.I`, `.cxx`); `BinType` enum from `panda/src/pgraph/cullBinEnums.h`
**Inherits:** CullBinEnums

Global registry (singleton via `get_global_ptr()`) of every named cull bin in the world: what type each bin is, its draw sort order, and whether it's active. [CullTraverser](CullTraverser.md)/[CullResult](CullResult.md) look up a bin by name during traversal (creating it via `make_new_bin()` on first use) and draw bins in ascending sort order once culling finishes.

## BinType (folded from `cullBinEnums.h`)

`CullBinEnums::BinType` is a tiny scoping-only base class shared by [CullBin](CullBin.md) and `CullBinManager` (too thin to warrant its own file):

| Value | Meaning |
|---|---|
| `BT_invalid` | Sentinel for "no such bin" / unrecognized name |
| `BT_unsorted` | Objects drawn in the order encountered, no sorting |
| `BT_state_sorted` | Sorted by RenderState to minimize state changes (used for opaque geometry) |
| `BT_back_to_front` | Sorted by distance from camera, farthest first (used for transparency) |
| `BT_front_to_back` | Sorted by distance from camera, nearest first |
| `BT_fixed` | Objects drawn in a fixed, externally-specified order |

`operator<<(ostream&, BinType)` prints the lowercase name (`"unsorted"`, `"state_sorted"`, …), matching the string tokens accepted by the `cull-bin` config variable.

## Behavior notes

- Constructed lazily as a singleton (`get_global_ptr()`); the constructor calls `setup_initial_bins()`, which first parses any `cull-bin <name> <sort> <type>` lines from the Config.prc file (via a `ConfigVariableList`), then adds five defaults **only if not already defined by name**: `background` (fixed, sort 10), `opaque` (state_sorted, sort 20), `transparent` (back_to_front, sort 30), `fixed` (fixed, sort 40), `unsorted` (unsorted, sort 50). This is why application code can safely assume these five bins always exist.
- `add_bin()` on an existing name is idempotent *only* if type and sort match exactly; a conflicting redefinition logs an error and returns -1 rather than overwriting.
- Bin indices are reused: `remove_bin()` marks a slot free rather than compacting the vector, and `add_bin()` will reclaim a freed slot before growing. Removed bin indices are NOT immediately safe to forget — `remove_bin()` explicitly calls `RenderState::bin_removed(bin_index)` and `CullResult::bin_removed(bin_index)` so every cached bin-index reference in the world (RenderState's cached `CullBinAttrib` lookups, live CullResults) gets invalidated. `remove_bin()` is documented as unsafe to call mid-frame — development/tooling use only.
- `get_num_bins()`/`write()` lazily re-sort `_sorted_bins` on demand (const methods internally `const_cast` to call `do_sort_bins()`) rather than eagerly re-sorting on every `set_bin_sort()` call.
- Concrete `CullBin` subclasses (in `panda/src/cull`) self-register their constructor for a `BinType` via `register_bin_type()`, typically at static-init time — this is why `pgraph` can create bins of a type it has no compile-time knowledge of.
- Flash-color getters/setters (`get_bin_flash_active()` etc.) are compiled out entirely in `NDEBUG` (release) builds.

## API

| Signature | Notes |
|---|---|
| `static CullBinManager *get_global_ptr()` | the singleton accessor |
| `int add_bin(name, BinType, sort)` | returns bin_index, or -1 on name conflict |
| `void remove_bin(int bin_index)` | dev/tooling only, not frame-safe |
| `int get_num_bins() const` / `int get_bin(int n) const` / `get_bins()` (MAKE_SEQ) | iterate bins in sorted (draw) order |
| `int find_bin(const std::string &name) const` | -1 if not found |
| `std::string get_bin_name(int bin_index) const` | |
| `BinType get_bin_type(int bin_index \| name) const` / `set_bin_type(...)` | change may apply next frame depending on bin type |
| `int get_bin_sort(int bin_index \| name) const` / `set_bin_sort(...)` | lower sort drawn first |
| `bool get_bin_active(int bin_index \| name) const` / `set_bin_active(...)` | inactive bin's geometry isn't rendered |
| `void write(std::ostream &out) const` | dumps name/type/sort per bin, one per line |
| `PT(CullBin) make_new_bin(int bin_index, gsg, draw_region_pcollector)` | for CullResult's use; dispatches to the registered `BinConstructor` |
| `void register_bin_type(BinType, BinConstructor *)` | called by concrete bin implementations at startup |

## Usage

```cpp
CullBinManager *bm = CullBinManager::get_global_ptr();
bm->add_bin("my-overlay", CullBinManager::BT_fixed, 100);
bm->set_bin_active("my-overlay", true);
```

## See also

- [CullBin](CullBin.md) — the object type this registry creates/tracks
- [CullResult](CullResult.md) — consumer that looks up bins by index during traversal
