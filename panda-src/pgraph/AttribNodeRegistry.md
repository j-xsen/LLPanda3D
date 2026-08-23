# AttribNodeRegistry

**Source:** `panda/src/pgraph/attribNodeRegistry.h` (+ `.I`, `.cxx`)
**Inherits:** (none — global singleton)

A process-global registry of `NodePath`s that scene-graph attribs may refer
to by reference — e.g. a `ClipPlaneAttrib` pointing at a `PlaneNode`, or a
`LightAttrib` pointing at a light node. Its purpose is to unify such
references across Bam-file loads: when a `.bam` file is loaded and it
contains an attrib that references a node by name+type, that reference is
first looked up here; if a registered node matches by name and node type,
the *registered* node is substituted for the one embedded in the bam file.
This lets, e.g., multiple loaded model files all end up sharing the
application's one canonical "sun" light node instead of each carrying its
own independent copy.

## Behavior notes

- Entries are keyed by `(TypeHandle, name)`, stored in an `ov_set<Entry>`
  ordered for fast lookup — `add_node()` records a `NodePath`'s type+name,
  `lookup_node(orig_node)` finds a registered node whose type+name match
  `orig_node` and returns it (or `orig_node` unchanged if nothing matches).
- This is opt-in bookkeeping: nothing populates it automatically — an
  application registers nodes it wants shared this way explicitly via
  `add_node()`, typically the canonical light/plane nodes attached
  somewhere permanent in the scene graph before loading models that
  reference them by name.

## API

| Method | Notes |
|---|---|
| `void add_node(const NodePath &attrib_node)` | Register a node by its current type+name |
| `bool remove_node(const NodePath &attrib_node)` | Unregister by NodePath |
| `void remove_node(int n)` / `void clear()` | Unregister by index / drop everything |
| `NodePath lookup_node(const NodePath &orig_node) const` | Resolve a bam-file-referenced node to its registered substitute, if any |
| `int find_node(const NodePath&) const` / `find_node(TypeHandle, name) const` | Index lookup |
| `int get_num_nodes() const` / `NodePath get_node(int n) const` / `get_nodes()` (MAKE_SEQ) | Enumeration |
| `TypeHandle get_node_type(int n) const` / `std::string get_node_name(int n) const` | Per-entry metadata |
| `static AttribNodeRegistry *get_global_ptr()` | The singleton |

## Usage

```cpp
NodePath render("render");
NodePath sun = render.attach_new_node(new DirectionalLight("sun"));
AttribNodeRegistry::get_global_ptr()->add_node(sun);
// Any later-loaded .bam referencing a DirectionalLight named "sun"
// resolves to this same NodePath instead of its own embedded copy.
```

## See also

[NodePath](NodePath.md), [BamFile](BamFile.md), [Loader](Loader.md).
