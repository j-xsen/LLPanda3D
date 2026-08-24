# InternalName

**Source:** `panda/src/gobj/internalName.h` (+ `.I`, `.cxx`); Python glue in `internalName_ext.h/.cxx` excluded (see module [README](README.md))
**Inherits:** TypedWritableReferenceCount (declared `final`)

An interned string identifier — `InternalName::make("foo")` always returns
the same shared pointer for the same string, so comparing two names is a
pointer-equality check rather than a string comparison. Used pervasively
wherever low-level Panda code needs cheap, hashable "keys": vertex column
names in a [GeomVertexColumn](GeomVertexColumn.md) (via
[GeomVertexArrayFormat](GeomVertexArrayFormat.md)), and shader input names
in [Shader](Shader.md). Same interning idea as `GeomVertexFormat`'s
`register_format()` described in the module README's "Copy-on-write and
interning" section, but keyed on a plain string rather than structural
content.

## Behavior notes

**Interning is a tree, not a flat table.** Every `InternalName` (except the
single root, `get_root()`) has a `_parent` and stores only its own
`_basename` — `get_name()` walks up to the root concatenating with `.`.
`make("foo.bar")` is equivalent to
`get_root()->append("foo")->append("bar")`: `append()` splits on the
*last* `.` and recurses, so building `"a.b.c"` interns `"a"`, then `"a.b"`,
then `"a.b.c"` as three separate, still-shared nodes — asking for `"a.b"`
elsewhere returns the same pointer as the intermediate node created while
interning `"a.b.c"`. Each node keeps its own `_name_table` (child
basename → child pointer, protected by its own `_name_table_lock`) rather
than there being one global table — lookup cost is proportional to name
depth, not to the total number of interned names.

**Literal-table fast path.** The templated `make(const char (&literal)[N])`
overload (selected implicitly for C-string-literal arguments) additionally
caches by the literal's *pointer identity* in a separate global
`_literal_table`, so repeated calls with the same string-pooled literal
(e.g. `InternalName::make("normal")` written at multiple call sites that
the compiler pools into one literal) skip even the tree-walking `append()`
lookup on subsequent calls. This is purely a speed optimization layered on
top of the string-based interning — it doesn't change identity semantics,
since the tree-based `append()` path is still what actually creates the
canonical instance the first time.

**Reference-count-triggered table cleanup.** `unref()` is overridden
(unusual — most Panda `ReferenceCount`-derived classes don't override
`unref()`) to hold the parent's `_name_table_lock` around the refcount
decrement and, if the count reaches zero, erase this name from its
parent's `_name_table` *before* the object is destructed — this closes a
race where another thread could look up the same basename in the interval
between the refcount hitting zero and the destructor removing the entry.
The destructor itself only asserts (debug builds) that the entry is
already gone, rather than removing it. The root `InternalName` (no parent)
skips this entirely and calls the base `unref()` directly.

**Bam (de)serialization does its own refcounting dance.** Because
`make_from_bam()` must return a raw `TypedWritable*` while going through
the interning table (which may hand back a pre-existing, already-owned
instance rather than constructing a fresh one owned by the caller), it
explicitly `ref()`s the returned name and registers a `finalize()`
callback that `unref()`s it once — the comment in the .cxx calls out that
using `unref_delete()` instead would be dangerous to call from within a
virtual function, and that hitting a zero refcount inside `finalize()`
would itself indicate a leak (nothing else is holding the pointer).
`make_texcoord_from_bam()` is a compatibility shim for bam files from
before this class was renamed from `TexCoordName` (versions 4.11–4.17).

## API

| Method | Notes |
|---|---|
| `make(name)` | Returns the shared instance for `name` (creating it if new); splits on `.` for hierarchical names. |
| `make(name, index)` | Convenience: concatenates `name` and `index` (e.g. `make("texcoord", 2)` → interns `"texcoord2"`). |
| `append(basename)` | Returns (creating if needed) the child of `this` named `basename` — cheaper than `make(get_name() + "." + basename)`. |
| `get_parent()` | This name's parent, or `nullptr` only for the root. |
| `get_name()` | Full dotted name from root to this node. |
| `join(sep)` | Like `get_name()` but with a custom separator instead of `.`. |
| `get_basename()` | Just this node's own segment (after the last `.`). |
| `find_ancestor(basename)` | Index of the nearest ancestor (0 = self) with the given basename, or -1. |
| `get_ancestor(n)` | The ancestor `n` levels up (0 = self); returns the root if `n` exceeds the chain length. |
| `get_top()` | The oldest non-root ancestor — e.g. `"texcoord.foo.bar"` → `"texcoord"`. |
| `get_net_basename(n)` | Basename prefixed by `n` levels of ancestors, joined with `.`. |
| `output(out)` | Writes the full dotted name (or `"(root)"`) to a stream. |

### Predefined stock names (all static, lazily interned on first call)

| Method | Interned string | Conventional meaning |
|---|---|---|
| `get_root()` | *(empty, no parent)* | Root of the whole interning tree. |
| `get_error()` | `"error"` | Sentinel for error conditions. |
| `get_vertex()` | `"vertex"` | Vertex position column. |
| `get_normal()` | `"normal"` | Lighting normal column. |
| `get_tangent()` / `get_tangent_name(name)` | `"tangent"` / `"tangent.name"` | Bump-mapping tangent vector, optionally per named texcoord set. |
| `get_binormal()` / `get_binormal_name(name)` | `"binormal"` / `"binormal.name"` | Bump-mapping binormal vector, ditto. |
| `get_texcoord()` / `get_texcoord_name(name)` | `"texcoord"` / `"texcoord.name"` | Default / named texture coordinate set — also identifies a [TextureStage](TextureStage.md)'s texcoord set. |
| `get_color()` | `"color"` | Per-vertex color column. |
| `get_rotate()` | `"rotate"` | Per-point-sprite rotation in degrees. |
| `get_size()` | `"size"` | Per-vertex point size override. |
| `get_aspect_ratio()` | `"aspect_ratio"` | Per-vertex point aspect ratio (x/y). |
| `get_transform_blend()` | `"transform_blend"` | Index into a [TransformBlendTable](TransformBlendTable.md) (CPU vertex animation). |
| `get_transform_weight()` | `"transform_weight"` | Hardware-skinning per-transform weight (GPU vertex animation). |
| `get_transform_index()` | `"transform_index"` | Hardware-skinning [TransformTable](TransformTable.md) index. |
| `get_morph(column, slider)` | `"column.morph.slider"` | Per-slider offset column for a morph/blend-shape target on `column`. |
| `get_index()` | `"index"` | Not a vertex data column — used by [GeomPrimitive](GeomPrimitive.md) to name the index array itself. |
| `get_world()` / `get_camera()` / `get_model()` / `get_view()` | `"world"` / `"camera"` / `"model"` / `"view"` | Shader-input keyword names (see [Shader](Shader.md)). |

## Usage

```cpp
CPT(InternalName) my_column = InternalName::make("my_custom_data");
if (my_column == InternalName::get_vertex()) {
  // pointer comparison, no string compare
}

// Hierarchical: interns "texcoord", then "texcoord.lightmap"
CPT(InternalName) lightmap_uv = InternalName::get_texcoord_name("lightmap");
```

## See also

- [GeomVertexColumn](GeomVertexColumn.md), [GeomVertexArrayFormat](GeomVertexArrayFormat.md) — use `InternalName` to identify vertex columns
- [Shader](Shader.md) — uses `InternalName` for shader input binding
- Module [README](README.md) — "Copy-on-write and interning" section for the related `GeomVertexFormat` interning pattern
