# GeomVertexFormat

**Source:** `panda/src/gobj/geomVertexFormat.h` (+ `.I`, `.cxx`)
**Inherits:** `TypedWritableReferenceCount` > `GeomVertexFormat` (also inherits `GeomEnums`, see [gobj README](README.md#shared-enums-geomenums)) — declared `final`
**Inherited by:** (none — final)

Defines the physical layout of the vertex data stored within a
[GeomVertexData](GeomVertexData.md): an ordered list of one or more
[GeomVertexArrayFormat](GeomVertexArrayFormat.md)s, each of which in turn
lists the named columns (`InternalName`s — see [InternalName](InternalName.md))
physically packed into that array. A column name must be unique across the
whole format, even between different arrays. This class is used for
constructing custom vertex data (choosing which attributes a mesh carries and
how they're packed) — most code, however, uses one of the built-in
stock formats.

## Behavior notes

- **Interned, not COW.** Unlike `GeomVertexData`/`GeomVertexArrayData`
  (copy-on-write, see [gobj README](README.md#copy-on-write-and-interning)),
  `GeomVertexFormat` is *interned*: `register_format()` looks up a global
  registry (`GeomVertexFormat::Registry`, keyed by content comparison via
  `IndirectCompareTo`) and returns the existing shared instance if an
  identical format is already registered, otherwise registers and returns
  a new one. A format **must be registered before use** — `GeomVertexData`'s
  constructor asserts `format->is_registered()`. Once registered, a format
  is treated as immutable; mutating methods (`add_array`, `remove_column`,
  etc.) assert `!_is_registered`. Comparison between two registered formats
  is therefore pointer-equality.
- **`unref()` is overridden** to unregister the format from the global
  registry when its refcount drops to zero while holding the registry lock
  — the same teardown pattern used by `GeomVertexArrayFormat` and, in
  `pgraph`, by `RenderState`/`TransformState` (see
  [pgraph README](../pgraph/README.md#the-state-pipeline)).
- **Stock formats** (`get_v3()`, `get_v3n3()`, `get_v3t2()`, `get_v3n3t2()`,
  `get_v3cp()`, `get_v3n3cp()`, `get_v3cpt2()`, `get_v3n3cpt2()`,
  `get_v3c4()`, `get_v3n3c4()`, `get_v3c4t2()`, `get_v3n3c4t2()`) are
  pre-registered singletons built once by `Registry::make_standard_formats()`.
  Naming convention: `v3`=3-component vertex position, `n3`=3-component
  normal, `t2`=2-component texcoord, `c4`=4-component OpenGL-style RGBA
  color (`NT_uint8` × 4), `cp`=DirectX-style packed color
  (`NT_packed_dcba`, one `Contents::C_color` column, 4 bytes total). Formats
  using `cp` "may not be supported directly by OpenGL" (the GL backend
  auto-converts, with runtime overhead); formats using `c4` are "not
  supported directly by DirectX" (the DX backend auto-converts, with larger
  overhead since older DirectX requires full interleaving) — pick based on
  target backend if it matters, otherwise either works everywhere via
  auto-conversion.
- **`get_animation()`** returns the format's `GeomVertexAnimationSpec` (see
  [GeomVertexAnimationSpec](GeomVertexAnimationSpec.md)) — whether/how this
  format encodes vertex animation. `get_post_animated_format()` returns the
  equivalent format *after* CPU-side animation has been applied (animation-
  specific columns like `transform_blend`/`transform_index`/`transform_weight`
  stripped out, since post-animation the result is static `C_point`/
  `C_normal`/`C_vector` data).
- **`get_union_format()`** computes a new registered format that is the
  superset of this format and another — used when merging geometry with
  different (but compatible) vertex layouts, e.g. during `SceneGraphReducer`
  flattening (see [pgraph README](../pgraph/README.md#loading-and-model-management)).
- Convenience accessors `get_vertex_column()`/`get_normal_column()`/
  `get_color_column()` (and their `*_array_index` counterparts) cache the
  lookup for the standard "vertex"/"normal"/"color" `InternalName`s, since
  those three are checked constantly by the render pipeline (e.g.
  `GeomVertexDataPipelineReader::has_vertex()`/`is_vertex_transformed()` in
  [GeomVertexData](GeomVertexData.md) use exactly this).
- `get_num_points()`/`get_num_vectors()`/`get_num_texcoords()`/
  `get_num_morphs()` expose format-wide indexes of columns by `Contents`
  category (`C_point`, `C_vector`, texcoord-named columns, and morph-delta
  triples respectively) — used internally by animation/transform code to
  know which columns need matrix-transforming vs. leaving alone.

## API

| Method | Notes |
|---|---|
| `GeomVertexFormat()` / `GeomVertexFormat(array_format)` / copy ctor | Unregistered by default — must call `register_format()` before handing to a `GeomVertexData`. |
| `bool is_registered() const` | — |
| `static CPT(GeomVertexFormat) register_format(format)` | Interns; also overloaded to accept a single `GeomVertexArrayFormat*` directly (wraps it in a 1-array format). |
| `const GeomVertexAnimationSpec &get_animation() const` / `set_animation(spec)` | Only settable pre-registration. |
| `CPT(GeomVertexFormat) get_post_animated_format() const` | See above. |
| `CPT(GeomVertexFormat) get_union_format(other) const` | See above. |
| `size_t get_num_arrays() const` / `get_array(i)` / `add_array(fmt)` / `insert_array(i, fmt)` / `set_array(i, fmt)` / `remove_array(i)` / `clear_arrays()` / `remove_empty_arrays()` | Array-list management, pre-registration only. |
| `size_t get_num_columns() const` / `get_column(i)` / `get_column(name)` / `has_column(name)` / `get_column_name(i)` | Column lookup across all arrays. |
| `int get_array_with(name) const` / `get_array_with(i)` | Which array index a given column (by name or column index) lives in. |
| `void remove_column(name, keep_empty_array=false)` | Pre-registration only. |
| `void pack_columns()` | Removes inter-column padding across all arrays. |
| `void align_columns_for_animation()` / `maybe_align_columns_for_animation()` | 16-byte-align `C_point`/`C_vector`/`C_normal` float columns for SSE2; the `maybe_` variant checks `vertex-animation-align-16` first. |
| `static const GeomVertexFormat *get_empty()` | Zero-column format. |
| `static const GeomVertexFormat *get_v3()` / `get_v3n3()` / `get_v3t2()` / `get_v3n3t2()` / `get_v3cp()` / `get_v3n3cp()` / `get_v3cpt2()` / `get_v3n3cpt2()` / `get_v3c4()` / `get_v3n3c4()` / `get_v3c4t2()` / `get_v3n3c4t2()` | Stock pre-registered formats, see Behavior notes. |
| `int get_vertex_array_index() const` / `get_vertex_column()` | And the `normal`/`color` equivalents. |
| `int compare_to(other) const` | Content-based ordering, used as the registry's comparator. |
| `void output(out) const` / `write(out, indent=0) const` / `write_with_data(out, indent, data) const` | Debug printing; the `_with_data` variant dumps actual vertex values too. |

## Usage

```cpp
PT(GeomVertexFormat) format = new GeomVertexFormat();
PT(GeomVertexArrayFormat) array_format = new GeomVertexArrayFormat();
array_format->add_column(InternalName::get_vertex(), 3, GeomVertexFormat::NT_float32, GeomVertexFormat::C_point);
array_format->add_column(InternalName::get_color(), 4, GeomVertexFormat::NT_packed_dabc, GeomVertexFormat::C_color);
format->add_array(array_format);
CPT(GeomVertexFormat) registered = GeomVertexFormat::register_format(format);

PT(GeomVertexData) vdata = new GeomVertexData("triangle", registered, Geom::UH_static);
```

## See also

- [gobj README](README.md) — shared enums, copy-on-write/interning overview
- [GeomVertexArrayFormat](GeomVertexArrayFormat.md), [GeomVertexColumn](GeomVertexColumn.md)
- [GeomVertexData](GeomVertexData.md) — the format's consumer
- [GeomVertexAnimationSpec](GeomVertexAnimationSpec.md)
- [InternalName](InternalName.md)
