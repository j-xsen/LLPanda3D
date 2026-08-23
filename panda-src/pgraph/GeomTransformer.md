# GeomTransformer

**Source:** `panda/src/pgraph/geomTransformer.h` / `.I` / `.cxx`

A vertex-data transformation helper used by [SceneGraphReducer](SceneGraphReducer.md)
and `NodePath`'s flatten/apply-transform operations. Its purpose is purely
about **sharing**: applying the same transform, texcoord remap, color
change, or vertex-collect operation to many `Geom`s that happen to share
the same underlying `GeomVertexData` should produce one new shared
`GeomVertexData`, not one duplicate per `Geom`. Geom/GeomVertexData
internals live in the `gobj` module (undocumented here) — this doc covers
only `GeomTransformer`'s role as a memoization layer over those types.

## Behavior notes

- Internally keyed by `(operation-specific-key, source GeomVertexData)` →
  new `GeomVertexData`, so calling e.g. `transform_vertices(geomA, mat)`
  and `transform_vertices(geomB, mat)` where `geomA`/`geomB` share vertex
  data produces one transformed `GeomVertexData` shared by both results,
  not two independent copies — this is what makes it safe/cheap to run a
  transform over an entire subtree.
- Distinguishes **"fixed" colors** (`set_color()` — a hard replacement, as
  from a `ColorAttrib`) from **"transformed" colors** (`transform_colors()`
  — scaled from the existing color, as from a `ColorScaleAttrib`) with
  separate internal caches (`_fcolors`/`_tcolors`), since the two are not
  interchangeable operations.
- `collect_vertex_data()`/`finish_collect()` implement the vertex-merging
  step of scene flattening: multiple `Geom`s' vertex data get combined into
  one larger buffer (subject to `max-collect-vertices`/`max-collect-indices`
  caps), reducing draw-call count. This is a two-phase API — call
  `collect_vertex_data()` per node, then `finish_collect()` once at the end
  to actually perform the merges.
- `premunge_geom()` produces a GSG-munged copy of a `Geom` via a
  `GeomMunger` (see [StateMunger](StateMunger.md)) — this is the mechanism
  behind the `premunge-data` config variable.

## API (grouped by purpose)

| Method | Purpose |
|---|---|
| `register_vertices(Geom/GeomNode*, might_have_unused)` | Pre-registers vertex data association bookkeeping |
| `transform_vertices(Geom/GeomNode*, const LMatrix4&)` | Bakes a matrix into vertex positions/normals |
| `transform_texcoords(Geom/GeomNode*, from, to, mat)` | Remaps one texcoord column through a matrix |
| `set_color(Geom/GeomNode*, LColor)` | Replaces vertex colors |
| `transform_colors(Geom/GeomNode*, LVecBase4 scale)` | Scales existing vertex colors |
| `apply_texture_colors(...)` | Bakes a texture's color (for a fixed/known TexMatrix+base color) into vertex colors |
| `apply_state(GeomNode*, const RenderState*)` | Bakes an entire node's state into geometry where possible |
| `set_format(Geom*, GeomVertexFormat*)` / `remove_column(Geom/GeomNode*, InternalName*)` | Vertex format changes |
| `make_compatible_state(GeomNode*)` | Normalizes state across a node's Geoms |
| `reverse_normals(Geom*)` / `doubleside(GeomNode*)` / `reverse(GeomNode*)` | Winding/normal flips |
| `collect_vertex_data(...)` / `finish_collect(bool format_only)` | Vertex-buffer merging (2-phase) |
| `finish_apply()` | Finalizes pending apply-state operations |
| `premunge_geom(const Geom*, GeomMunger*)` | GSG-specific vertex munging |
| `get_max_collect_vertices()` / `set_max_collect_vertices(int)` | Per-instance override of `max-collect-vertices` |

## See also

- [SceneGraphReducer](SceneGraphReducer.md) (primary caller), [StateMunger](StateMunger.md)
