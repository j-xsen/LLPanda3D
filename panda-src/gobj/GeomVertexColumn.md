# GeomVertexColumn

**Source:** `panda/src/gobj/geomVertexColumn.h` (+ `.I`, `.cxx`) — largest file in this cluster (~5.4k lines), almost entirely from the internal `Packer` class family, not the public interface.
**Inherits:** `GeomEnums` (see [gobj README](README.md#shared-enums-geomenums))
**Inherited by:** (none)

Describes one named field within a [GeomVertexArrayFormat](GeomVertexArrayFormat.md)
— its [InternalName](InternalName.md) (e.g. `"vertex"`, `"normal"`,
`"texcoord"`), numeric type, `Contents` semantic tag, component count, and
byte offset/stride within a row. A `GeomVertexArrayFormat` owns a list of
these. Application code rarely constructs one directly except when building
a custom `GeomVertexArrayFormat`; reading/writing the actual values it
describes goes through [GeomVertexReader/Writer/Rewriter](GeomVertexReader.md),
not this class.

## Behavior notes

- **Why it's huge:** the public surface is small (accessors + `set_*`
  mutators). The bulk of the file is the private `Packer` class hierarchy —
  ~25 specialized subclasses (`Packer_point_nativefloat_3`,
  `Packer_argb_packed`, `Packer_rgba_uint8_4`, `Packer_uint16_1`, …) that
  implement fast, type-specific pack/unpack of raw bytes to/from
  `LVecBase2/3/4{f,d,i}`. `setup()` (called once at construction/bam-load)
  picks the right `Packer` via `make_packer()`, switching on `Contents` and
  `NumericType`/component-count combination, and even distinguishing
  "native float matches `PN_float32`" vs. not, and 16-byte-aligned vs.
  unaligned starts, to select the fastest available path. `GeomVertexReader`/
  `Writer` delegate all actual data conversion to this column's `Packer`.
- **`Contents`-driven special-case reading, not just storage:**
  - `C_point`/`C_clip_point`/`C_texcoord` are read as 4-D homogeneous
    points: if the column only stores 2 or 3 components, the missing W is
    implicitly `1.0`; if it stores fewer components than requested on read
    (e.g. `get_data3f()` on a 2-component column), and W *is* present,
    3-or-fewer-component reads implicitly divide by W (`Packer_point`).
  - `C_color` implicitly treats a missing 4th (alpha) component as `1.0`
    (never divided, unlike points), and integer color storage
    (`NT_uint8`, `NT_packed_dabc`) is transparently rescaled to/from the
    `0.0–1.0` float range on read/write (`Packer_color`). `C_color` columns
    must have 3 or 4 values — `make_packer()` logs a `gobj_cat.error()` if
    not (but still returns a working generic `Packer_color`, doesn't abort).
  - `C_normal` likewise requires 3 or 4 values (error-logged otherwise) but
    is NOT auto-scaled like color, since it just falls through to the
    generic numeric packer.
  - `NT_matrix`'s `_num_elements` default is special-cased in `setup()`:
    for `Contents::C_matrix`, `_num_elements` defaults to `_num_components`
    (i.e. an N-component matrix column is assumed square, N×N), instead of
    the usual default of `1`.
- **`is_packed_argb()` / `is_uint8_rgba()`** are quick predicates used
  elsewhere (e.g. `GeomVertexData`'s `packed_argb_to_uint8_rgba()`/
  `uint8_rgba_to_packed_argb()` bulk conversion helpers) to fast-path the
  two common DirectX-vs-OpenGL color encodings without going through the
  generic `Packer`.
- **Alignment defaults:** if `column_alignment` isn't specified explicitly,
  `setup()` defaults it to `max(component_byte_size, vertex-column-alignment
  config var)`, then rounds `_start` up to that alignment — so two columns
  built with the same nominal `start` can end up at different actual
  offsets depending on their numeric type.
- **Bam compatibility shim:** reading a pre-1.10-minor-version-38 `.bam`
  file remaps `Contents::C_vector` back to `C_normal` for a column named
  `"normal"` (in `complete_pointers()`) — `C_normal` didn't exist as a
  distinct value in Panda 1.9 and earlier, so old files stored normals as
  plain vectors; the reverse remap happens in `write_datagram()` when
  writing for an old-enough target version.

## API

| Method | Notes |
|---|---|
| `GeomVertexColumn(name, num_components, numeric_type, contents, start, column_alignment=0, num_elements=0, element_stride=0)` | Primary constructor, called by `GeomVertexArrayFormat::add_column()`. |
| `const InternalName *get_name() const` | — |
| `int get_num_components() const` / `get_num_values() const` | `num_values` counts scalar values after unpacking (e.g. a packed-color column has `num_components=1` but `num_values=4`). |
| `int get_num_elements() const` / `get_element_stride() const` | For array-valued columns (e.g. matrix columns), the element count and per-element byte stride. |
| `NumericType get_numeric_type() const` / `Contents get_contents() const` | — |
| `int get_start() const` / `get_column_alignment() const` | Byte offset within the row, and required alignment. |
| `int get_component_bytes() const` / `get_total_bytes() const` | Per-component byte size, and total bytes this column occupies. |
| `bool has_homogeneous_coord() const` | True for `C_point`/`C_clip_point`/`C_texcoord` types that carry an implicit/explicit W. |
| `bool overlaps_with(start_byte, num_bytes) const` | Byte-range overlap test, used by `GeomVertexArrayFormat::add_column()`'s conflict eviction. |
| `bool is_bytewise_equivalent(other) const` | Same physical layout regardless of semantic name — used by `is_data_subset_of()`. |
| `void set_name(name)` / `set_num_components(n)` / `set_numeric_type(t)` / `set_contents(c)` / `set_start(n)` / `set_column_alignment(n)` | Mutators — only meaningful before the owning format is registered. |
| `bool is_packed_argb() const` / `is_uint8_rgba() const` | Fast-path predicates for the two common color encodings. |
| `int compare_to(other) const` / `operator==` / `operator!=` / `operator<` | Content-based ordering. |
| `void output(out) const` | Compact one-line format (e.g. `vertex(3f)`, `color(1p)` for a packed 4-byte color). |

## Usage

Application code almost never touches `GeomVertexColumn` directly beyond
looking one up:

```cpp
const GeomVertexColumn *col = format->get_column(InternalName::get_vertex());
if (col != nullptr) {
  nout << "vertex column: " << col->get_num_components()
       << " components, " << col->get_total_bytes() << " bytes\n";
}
```

## See also

- [gobj README](README.md) — shared enums
- [GeomVertexArrayFormat](GeomVertexArrayFormat.md) — owns a list of these
- [GeomVertexReader](GeomVertexReader.md) / GeomVertexWriter / GeomVertexRewriter — actual data access, delegates to this column's `Packer`
- [InternalName](InternalName.md)
