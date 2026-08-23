# NodePath

**Source:** `panda/src/pgraph/nodePath.h` (+ `.I`, `.cxx`)

The class application code actually uses to work with the scene graph (see
[module README](README.md#nodepath-vs-pandanode) for why a bare
[PandaNode](PandaNode.md) isn't enough). A `NodePath` is a lightweight,
copy-by-value list of connected nodes from some root down to a specific
node, backed by a chain of ref-counted
[NodePathComponent](NodePathComponent.md)s stored on the nodes themselves —
holding a `NodePath` keeps its nodes alive, and if a node in the chain is
reparented (even via a different `NodePath`), every `NodePath` referencing
it updates automatically since they all share the same component objects.

`nodePath_ext.h/.I/.cxx` (Python `__deepcopy__`/`__reduce__`/pickling glue)
is excluded — Python-only.

## Behavior notes

- **Validity / emptiness**: `is_empty()` includes both "never assigned" and
  error states (`get_error_type()`: `ET_not_found` from a failed `find()`,
  `ET_removed` after `remove_node()`, `ET_fail` for general failure). Check
  `get_error_type()` when a `NodePath`-returning call might have failed,
  rather than assuming any non-crash result is valid.
- **`get_key()`** returns a process-lifetime-unique int identifying this
  specific instance (delegates to
  `NodePathComponent::get_key()`); useful as a map key when a
  pointer-comparable, stable identity for "this occurrence of this node" is
  needed. `is_singleton()` means the path is exactly one node long (no
  parent component).
- **Search path strings** (`find()`, `find_all_matches()`, etc.) use a
  compiled-once-per-call [FindApproxPath](FindApproxPath.md) grammar: plain
  names, `*` (one node, any name), `**` (zero or more nodes),
  `+TypeName`/`-TypeName` (inexact/exact type match), `=tag`/`=tag=value`
  (tag match), `@@` prefix (stashed-only match), standard glob characters
  in names, and a trailing `;+h-s-i`-style flag suffix (return-hidden /
  return-stashed / case-insensitive). Ambiguous matches prefer the
  *shortest* path; `find_all_matches` sorts shortest-first.
- **reparent_to vs. wrt_reparent_to vs. instance_to vs. copy_to**:
  - `reparent_to(other)` moves the node (detach + attach) and **resets the
    node's prev-transform delta** (so motion-blur/velocity tracking doesn't
    see a bogus jump). It leaves the node's local transform value
    unchanged — the node visually jumps to wherever `other`'s coordinate
    space puts it.
  - `wrt_reparent_to(other)` does the same move but first recomputes the
    node's transform relative to `other` so the node's **world-space
    position doesn't change** across the reparent (an implicit `wrt()`).
  - `instance_to(other)` does **not** remove the node from its existing
    parent(s) — it creates an additional graph instance under `other` and
    returns a `NodePath` to that new instance. If the node is already
    instanced under `other`, it returns (and unstashes, if needed) that
    existing instance rather than creating a duplicate. Unlike the reparent
    variants, it does **not** reset the prev-transform delta.
  - `copy_to(other)` is like `instance_to()` but deep-copies the node and
    its whole subgraph first (via `PandaNode::copy_subgraph()`), so the
    original is left completely untouched.
  - `attach_new_node(node_or_name)` creates/attaches a brand new node as a
    child — the ordinary way to add new geometry/structure.
  - `stash_to(other)` = `reparent_to(other)` immediately followed by
    `stash()`.
- **`is_ancestor_of()`/`get_common_ancestor()`** both rely on an internal
  `find_common_ancestor()` walk that also verifies the two paths are in the
  same graph — an implicit `wrt()`-style relative-transform call
  (`get_transform(other)`, `get_pos(other)`, etc.) between unrelated graphs
  fails unless `allow-unrelated-wrt` (see module README config vars) is
  set, in which case it falls back to treating the render root as the
  common point.
- **Convenience transform/state setters follow one pattern**: an unqualified
  form (`set_pos(x,y,z)`, `set_color(...)`, `set_texture(...)`, …) reads/
  writes the node's own local `RenderState`/`TransformState` slot directly;
  a second overload taking a leading `const NodePath &other` performs the
  equivalent operation **relative to `other`'s coordinate space** (an
  implicit `wrt()` for transform setters) instead of local space. Most
  state-affecting setters also take a trailing `int priority = 0`, which
  feeds `RenderAttrib` override-priority (higher priority wins when
  composing state down the graph — see
  [RenderState](RenderState.md)/[RenderAttrib](RenderAttrib.md)). There's a
  `set_fluid_*` variant of the position setters that skips the implicit
  velocity-delta update `set_pos` otherwise performs — use it for teleports
  where you don't want the object's "previous position" to register a
  giant jump for motion blur.
- **`show()`/`hide()`/`show_through()`** take an optional per-camera
  `DrawMask`; with none given they affect the special "overall" bit (hidden
  from every camera). See `PandaNode::adjust_draw_mask` in
  [PandaNode.md](PandaNode.md) for the underlying show/hide/clear-mask
  semantics.
- **Flattening tiers** (`flatten_light/medium/strong`), all built on
  [SceneGraphReducer](SceneGraphReducer.md): `flatten_light()` only bakes
  attribs (transforms/colors/tex matrices) into vertices, removing zero
  nodes. `flatten_medium()` additionally removes redundant single-child
  grouping nodes. `flatten_strong()` goes further and combines sibling
  nodes together where possible — the module README's config vars
  (`preserve-geom-nodes`, `flatten-geoms`, `max-collect-vertices/indices`)
  tune the geometry-combining passes. Strong flattening can destroy
  hierarchical bounding-volume structure, so it's recommended only for
  subtrees that will always be culled as one unit (a car, a prop), not
  whole scenes.
- **`verify_complete()`** (called internally by most mutating methods)
  checks the path's component chain is still fully connected — catches the
  case where an intermediate node was detached out from under a `NodePath`
  that still references something below it.

## API (grouped; individual convenience setters follow the patterns above)

**Construction / validity**
| | |
|---|---|
| `NodePath()` / `NodePath(name)` / `NodePath(PandaNode*)` / `NodePath(parent, child_node)` | |
| `any_path(node)` | Static: build a NodePath via an arbitrary (possibly ambiguous) instance of `node` |
| `not_found()` / `removed()` / `fail()` | Static error-flagged empty NodePaths |
| `is_empty()` / `operator bool()` / `get_error_type()` | |
| `is_singleton()` / `get_num_nodes()` / `get_node(i)` / `get_ancestor(i)` | |
| `get_top_node()` / `get_top()` | Root PandaNode / root NodePath |
| `node()` | The referenced `PandaNode*` |
| `get_key()` | Process-unique instance id |
| `is_same_graph(other)` / `is_ancestor_of(other)` / `get_common_ancestor(other)` | |

**Graph structure**
| | |
|---|---|
| `get_children()` / `get_num_children()` / `get_child(n)` / `get_stashed_children()` | |
| `count_num_descendants()` |  |
| `has_parent()` / `get_parent()` / `get_sort()` | |
| `find(path)` / `find_path_to(node)` / `find_all_matches(path)` / `find_all_paths_to(node)` | See search-path grammar above |

**Graph mutation**
| | |
|---|---|
| `reparent_to` / `wrt_reparent_to` / `stash_to` | Move, see Behavior notes |
| `instance_to` / `instance_under_node` / `copy_to` | Add-without-removing / deep-copy, see Behavior notes |
| `attach_new_node(node_or_name, sort=0)` | New child |
| `remove_node()` / `detach_node()` | Delete vs. detach-but-keep-alive-via-this-NodePath |
| `stash()` / `unstash()` / `unstash_all()` / `is_stashed()` / `get_stashed_ancestor()` | |
| `show()` / `show_through()` / `hide()` / `is_hidden()` / `get_hidden_ancestor()` | |

**Transform** (each has a `set_x(...)`/`get_x(...)` local form and an
`set_x(other, ...)`/`get_x(other, ...)` relative-to-`other` form)
| | |
|---|---|
| `set_pos` / `set_fluid_pos` / `get_pos` / `get_pos_delta` | |
| `set_hpr` / `get_hpr` / `set_quat` / `get_quat` | |
| `set_scale` / `get_scale` / `set_shear` / `get_shear` | |
| `set_pos_hpr[_scale][_shear]` / `set_pos_quat[_scale][_shear]` combos | Compound setters |
| `set_mat` / `get_mat` / `has_mat` / `clear_mat` | Raw `LMatrix4` |
| `look_at` / `heads_up` | |
| `get_relative_point` / `get_relative_vector` / `get_distance` | |

**Render state (local + `get_x(other)`/relative forms where noted)**
| | |
|---|---|
| `get_state()` / `set_state()` / `get_net_state()` | Whole RenderState |
| `set_attrib` / `get_attrib` / `has_attrib` / `clear_attrib` | Single RenderAttrib by type |
| `set_effect` / `get_effect` / `set_effects` / `get_effects` | |
| `set_color` / `set_color_scale` / `compose_color_scale` / `set_alpha_scale` | |
| `set_light` / `set_light_off` / `clear_light` / `has_light` | Attach a light-casting NodePath |
| `set_clip_plane` / `clear_clip_plane` / `has_clip_plane` | |
| `set_scissor` / `clear_scissor` / `has_scissor` | |
| `set_occluder` / `clear_occluder` / `has_occluder` | |
| `set_bin(name, order, priority)` / `clear_bin` / `get_bin_name` / `get_bin_draw_order` | Cull bin assignment |
| `set_texture` / `set_texture_off` / `clear_texture` / `get_texture` / `replace_texture` | Per-TextureStage overloads throughout |
| `set_shader` / `set_shader_off` / `set_shader_auto` / `clear_shader` | |
| `set_shader_input(...)` | Many overloads — textures, buffers, NodePaths, PTA arrays, vectors/matrices, ints/floats |
| `set_instance_count` / `get_instance_count` | Hardware instancing count |
| `set_tex_transform` / `set_tex_offset` / `set_tex_rotate` / `set_tex_scale` / `set_tex_pos` / `set_tex_hpr` | Per-stage texture matrix, local + relative |
| `set_tex_gen` / `clear_tex_gen` / `has_tex_gen` | Auto texture-coord generation |
| `set_tex_projector` / `clear_tex_projector` / `project_texture` | |
| `find_texture` / `find_all_textures` / `find_texture_stage` / `find_all_texture_stages` / `unify_texture_stages` | |
| `set_material` / `clear_material` / `get_material` / `find_material` / `find_all_materials` / `replace_material` | |
| `set_fog` / `clear_fog` / `get_fog` | |
| `set_render_mode*` (wireframe/filled/thickness/perspective) | |
| `set_two_sided` / `set_depth_test` / `set_depth_write` / `set_depth_offset` | |
| `set_billboard_axis` / `set_billboard_point_eye` / `set_billboard_point_world` / `do_billboard_*` / `clear_billboard` | `do_billboard_*` compute once immediately; `set_billboard_*` attach a live [BillboardEffect](BillboardEffect.md) |
| `set_compass` / `clear_compass` | |
| `set_transparency` / `set_logic_op` / `set_antialias` / `set_audio_volume` | |
| `adjust_all_priorities(delta)` | Bulk-shift RenderAttrib priorities at and below this node |

**Tags / bounds / misc**
| | |
|---|---|
| `set_tag` / `get_tag` / `has_tag` / `clear_tag` / `get_net_tag` / `find_net_tag` | `net_*` variants search up through ancestors |
| `get_bounds()` / `force_recompute_bounds()` / `calc_tight_bounds()` / `show_bounds()` / `hide_bounds()` | |
| `get_collide_mask()` / `set_collide_mask()` | Batch over this node + descendants |
| `flatten_light()` / `flatten_medium()` / `flatten_strong()` / `apply_texture_colors()` / `clear_model_nodes()` | See Behavior notes |
| `premunge_scene(gsg)` / `prepare_scene(gsg)` | Pre-upload geometry/state to a GSG |
| `write_bam_file` / `write_bam_stream` / `encode_to_bam_stream` / `decode_from_bam_stream` | |
| `ls()` / `reverse_ls()` / `output()` | Debug dump (down / up the tree) |

## Usage

```cpp
NodePath render("render");
NodePath model = window->load_model(render, "panda.egg");
model.set_pos(0, 10, 0);
model.set_scale(0.5);
model.set_color(1, 0, 0, 1);

NodePath light_np = render.attach_new_node(new DirectionalLight("sun"));
model.set_light(light_np);

NodePathCollection heads = model.find_all_matches("**/head");
```

## See also

[PandaNode](PandaNode.md), [NodePathComponent](NodePathComponent.md),
[NodePathCollection](NodePathCollection.md),
[WeakNodePath](WeakNodePath.md), [FindApproxPath](FindApproxPath.md),
[RenderState](RenderState.md), [TransformState](TransformState.md)
