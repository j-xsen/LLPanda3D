# SceneGraphAnalyzer

**Source:** `panda/src/pgraphnodes/sceneGraphAnalyzer.h` (+ `.I`, `.cxx`)
**Inherits:** (none — standalone)
**Inherited by:** (none)

**Not a scene graph node**, despite the module it lives in — a standalone
utility that walks a scene graph subtree and collects statistics: node/
instance/transform counts, `Geom`/vertex/triangle/texture counts and
memory usage, and redundancy detection (duplicated `GeomVertexData`/
`GeomVertexArrayData`). Used for scene profiling/auditing (e.g. an editor's
"scene info" panel, or a build-time asset-bloat check), not at runtime
during rendering.

## Behavior notes

- **`add_node()` accumulates rather than resets.** Calling it more than
  once (without an intervening `clear()`) merges statistics for multiple
  subgraphs into one combined report — useful for analyzing several
  independent scenes together. Call `clear()` first to start a fresh
  count.
- **Instance detection is identity-based and stateful across the whole
  traversal.** A `pmap<PandaNode*, int>` (`_nodes`) records every distinct
  node pointer visited; the second and later times the *same* node pointer
  is reached (i.e. it's parented in more than one place — a real Panda3D
  instance, not just similar content), it's counted as `_num_instances`
  and everything below it is marked `under_instance` so its own
  descendants aren't double-counted as additional top-level nodes in
  per-node stats (though their vertex/triangle counts are still
  accumulated normally, since the geometry really is drawn multiple
  times).
- **Two different kinds of "duplicate" are tracked, and they mean
  different things.** `_vdatas`/`_vadatas` (a `pset`) key by pointer
  identity — the same shared `CPT(GeomVertexData)` referenced from
  multiple `Geom`s is naturally counted once here, so `write()`'s vertex/
  triangle counts aren't inflated by legitimate sharing. `_unique_vdatas`/
  `_unique_vadatas` (a `pmap` keyed with `IndirectCompareTo`, i.e.
  content/byte comparison rather than pointer comparison) separately
  detects *distinct objects with identical content* — this is genuine
  waste (the same data needlessly duplicated in memory instead of shared),
  reported by `write()` as "N GeomVertexDatas are redundantly duplicated"
  and "N GeomVertexArrayDatas are redundant, wasting NK".
- **`LodMode` controls how `LODNode` subtrees are analyzed** — `LM_all`
  (default) walks every switch level's children (so stats reflect all LOD
  variants combined, typically the useful default for a memory audit),
  `LM_lowest`/`LM_highest` walk only the child at
  `get_lowest_switch()`/`get_highest_switch()` (see
  [LODNode](LODNode.md)), and `LM_none` stops at the `LODNode` itself
  without descending into any switch level at all.
- Normal-vector length is tracked (`_num_long_normals`/
  `_num_short_normals`/`_total_normal_length`) as a data-quality check —
  normals that aren't close to unit length (`IS_NEARLY_EQUAL(length,
  1.0f)`) usually indicate a modeling/export bug.
- `write()` is the only way to get a human-readable report; there's no
  structured/JSON export — for programmatic access, use the individual
  `get_num_*()`/`get_*_bytes()` accessors instead.

## API

| Method | Notes |
|---|---|
| `SceneGraphAnalyzer()` | Construct with `LM_all` LOD mode. |
| `clear()` | Reset all accumulated statistics. |
| `add_node(PandaNode *node)` | Recursively walk this subtree, accumulating into the current totals. Can be called multiple times to combine subgraphs. |
| `write(std::ostream &out, int indent_level = 0) const` | Print a human-readable multi-line report. |
| `set_lod_mode(LodMode)` / `get_lod_mode()` | `LM_lowest` / `LM_highest` / `LM_all` (default) / `LM_none` — see Behavior notes. |
| `get_num_nodes/instances/transforms/nodes_with_attribs/lod_nodes/geom_nodes/geoms()` | Coarse structural counts. |
| `get_num_geom_vertex_datas/formats()`, `get_vertex_data_size()` | `GeomVertexData`/`Format` counts and total unique array memory (bytes). |
| `get_num_vertices/normals/colors/texcoords()` | Column presence-weighted counts (a `GeomVertexData` only contributes to `get_num_normals()` if it actually has a normal column, etc). |
| `get_num_tris/lines/points/patches()` | Total primitive counts by coarse topology. |
| `get_num_individual_tris/tristrips/triangles_in_strips/trifans/triangles_in_fans()` | Triangle breakdown by how they're encoded (loose triangles vs. strip/fan-encoded). |
| `get_num_vertices_in_patches()` | Total patch-control-point count. |
| `get_texture_bytes()` | Estimated minimum GPU texture memory across all distinct `Texture`s referenced. |
| `get_num_long_normals/short_normals()`, `get_total_normal_length()` | Non-unit-length normal detection (data-quality check). |

## Usage

```cpp
SceneGraphAnalyzer analyzer;
analyzer.add_node(scene_root.node());
analyzer.write(std::cout);
std::cout << "Triangles: " << analyzer.get_num_tris() << "\n";
```

## See also

- [LODNode](LODNode.md) — `LodMode` interacts with `get_lowest_switch()`/
  `get_highest_switch()`
- [../gobj/GeomVertexData.md](../gobj/GeomVertexData.md),
  [../gobj/GeomVertexArrayData.md](../gobj/GeomVertexArrayData.md) — the
  objects whose sharing/duplication this class detects
