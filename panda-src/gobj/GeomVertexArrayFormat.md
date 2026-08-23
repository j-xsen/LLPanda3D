# GeomVertexArrayFormat

**Source:** `panda/src/gobj/geomVertexArrayFormat.h` (+ `.I`, `.cxx`)
**Inherits:** `TypedWritableReferenceCount` > `GeomVertexArrayFormat` (also inherits `GeomEnums`, see [gobj README](README.md#shared-enums-geomenums)) — declared `final`
**Inherited by:** (none — final)

Describes the structure of a single physical array within a
[GeomVertexFormat](GeomVertexFormat.md) (which is a list of these). A
`GeomVertexArrayFormat` holds an ordered list of
[GeomVertexColumn](GeomVertexColumn.md)s, each naming a field (by
[InternalName](InternalName.md)) and giving its byte offset, numeric type,
and semantic contents within one row of the array's byte stride. Often a
`GeomVertexFormat` has just one array with all columns interleaved
("vertex"/"normal"/"texcoord"/"color" side by side per row, DirectX-8-style);
it may instead split columns across multiple parallel arrays.

## Behavior notes

- **Interned exactly like `GeomVertexFormat`.** `register_format()` looks up
  a global `Registry` (comparison-keyed `pset`) and returns a shared
  instance for identical content; `unref()` is overridden to unregister
  from that registry when the refcount hits zero (same teardown pattern as
  `GeomVertexFormat` — see its doc's Behavior notes). Mutating methods
  (`add_column`, `remove_column`, `set_stride`, …) assert `!_is_registered`.
- **`add_column()` quietly evicts conflicts:** adding a column with the same
  name as an existing one, or that byte-overlaps one or more existing
  columns, silently `remove_column()`s the conflicting column(s) first — no
  error, no warning. `remove_column()` leaves a gap in the byte layout
  rather than compacting; call `pack_columns()` afterward to remove wasted
  space (it works by capturing all columns, `clear_columns()`, and
  re-`add_column()`-ing each in original order with `start = -1`, i.e.
  auto-placed immediately after the previous column).
- **Alignment:** `_stride`/`_total_bytes`/`_pad_to` are recomputed on every
  `add_column()`. `align_columns_for_animation()` re-adds every `C_point`/
  `C_vector`/`C_normal` float32/float64 column (≥3 components) at a forced
  16-byte alignment (padding to 4 components) for SSE2; other columns are
  re-added unchanged. This is a real behavior gotcha: calling it can widen
  a column's component count (3→4) as a side effect of the alignment pass.
- Global default column byte-alignment comes from the `vertex-column-alignment`
  config var (see [gobj README](README.md#config-variables-from-config_gobjh-cxx)).

## API

| Method | Notes |
|---|---|
| `GeomVertexArrayFormat()` | Unregistered by default. |
| `GeomVertexArrayFormat(name0, n0, type0, contents0, [name1, ...] up to 4 columns)` | Convenience constructors for 1–4 columns at once (auto-placed). |
| `bool is_registered() const` | — |
| `static CPT(GeomVertexArrayFormat) register_format(format)` | Interns. |
| `int get_stride() const` / `set_stride(n)` | Bytes per row; auto-grows on `add_column()`, can be widened manually (e.g. to reserve trailing padding). |
| `int get_pad_to() const` / `set_pad_to(n)` | Row alignment requirement. |
| `int get_divisor() const` / `set_divisor(n)` | Instancing divisor (0 = per-vertex, >0 = per-N-instances) for hardware instanced rendering. |
| `int get_total_bytes() const` | Bytes actually used by columns (≤ stride). |
| `int add_column(name, num_components, numeric_type, contents, start=-1, column_alignment=0)` | `start=-1` auto-places after the last column. Returns new column's index. |
| `int add_column(const GeomVertexColumn &column)` | Add a pre-built column descriptor. |
| `void remove_column(name)` / `clear_columns()` / `pack_columns()` / `align_columns_for_animation()` | See Behavior notes. |
| `int get_num_columns() const` / `get_column(i)` / `get_column(name)` / `get_column(start_byte, num_bytes)` / `has_column(name)` | Lookup, including by byte-range overlap. |
| `bool is_data_subset_of(other) const` | True if every column here has a bytewise-equivalent match in `other` at the same stride — used to check whether a conversion is a true no-op. |
| `int count_unused_space() const` | Padding bytes not covered by any column. |
| `void output(out) const` / `write(out, indent=0) const` / `write_with_data(out, indent, array_data) const` | Debug printing. |
| `std::string get_format_string(pad=true) const` | Human-readable one-line summary (used in log messages, e.g. by `GeomVertexData::convert_to()`'s debug output). |

## Usage

```cpp
PT(GeomVertexArrayFormat) array_format = new GeomVertexArrayFormat();
array_format->add_column(InternalName::get_vertex(), 3, GeomVertexFormat::NT_float32, GeomVertexFormat::C_point);
array_format->add_column(InternalName::get_normal(), 3, GeomVertexFormat::NT_float32, GeomVertexFormat::C_normal);
array_format->add_column(InternalName::get_texcoord(), 2, GeomVertexFormat::NT_float32, GeomVertexFormat::C_texcoord);
// Wrap in a GeomVertexFormat and register that, per GeomVertexFormat.md.
```

## See also

- [gobj README](README.md) — shared enums, interning overview
- [GeomVertexFormat](GeomVertexFormat.md) — owns a list of these
- [GeomVertexColumn](GeomVertexColumn.md)
- [GeomVertexArrayData](GeomVertexArrayData.md) — the physical data matching this layout
