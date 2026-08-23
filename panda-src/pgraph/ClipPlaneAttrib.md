# ClipPlaneAttrib

**Source:** `panda/src/pgraph/clipPlaneAttrib.h` (+ `.I`, `.cxx`)
**Inherits:** [RenderAttrib](RenderAttrib.md)

Tracks the set of [PlaneNode](PlaneNode.md)-based clipping planes in effect
at this level and below, functioning like [LightAttrib](LightAttrib.md) but
for clip planes: it holds independent "on" and "off" lists (each a
`NodePath` pointing at a `PlaneNode`) rather than a single replace-or-merge
operation, so a subtree can turn specific inherited planes back off while
adding others.

## Behavior notes
- Each plane is stored as a `NodePath` (not a bare `PlaneNode*`) since the
  same `PlaneNode` may appear at multiple graph locations with different
  transforms — the attrib needs to know *which instance*.
- `make()` (no on/off planes — identity) and `make_all_off()` each cache a
  single shared singleton (`_empty_attrib`/`_all_off_attrib`), found once
  and reused.
- `has_all_off()`: true if this attrib turns off every previously-on plane
  (not just the ones listed in `_off_planes`); `has_off_plane()` accounts
  for this by also checking `_off_all_planes && !has_on_plane(plane)`.
- `filter_to_max(n)`: returns a `ClipPlaneAttrib` reduced to at most `n` on
  planes, keeping the highest-`PlaneNode::get_priority()` ones. Results
  are memoized per `n` in `_filtered`, invalidated automatically whenever
  any `PlaneNode`'s priority changes (tracked via a global
  `PlaneNode::get_sort_seq()` `UpdateSeq`, checked by `check_filtered()`).
- `compose_off()` is a special internal composition (used by
  `PandaNode::get_off_clip_planes()`) that unions only the *off* lists of
  two attribs, ignoring the on lists — used to accumulate "which planes
  are forbidden below here" independently of what's currently on.
- `compose_impl`: if `other` turns off all planes, it wins outright.
  Otherwise a plane on in `other`'s off-list cancels a plane on in this
  attrib's on-list (three-way sorted merge of on/on/off).
  `invert_compose_impl` simply returns `other` — the comment in the source
  admits this is likely not fully correct but not worth the complexity to
  fix.
- The old `Operation`-based interface (`make(op, plane...)`,
  `get_operation()`, `get_num_planes()`, `get_plane()`, `has_plane()`,
  `add_plane()`, `remove_plane()`) is **deprecated** — logs a warning on
  every call and internally just dispatches to the on/off-plane interface
  below. New code should not use it; omitted from the API table.
- Bam versioning: pre-4.0-minor-version files stored raw node pointers and
  reconstruct `NodePath`s from `AttribNodeRegistry` in `finalize()`; 4.0+
  files store full `NodePath`s directly via `NodePath::write_datagram()`.

## API

| Method | Notes |
|---|---|
| `static CPT(RenderAttrib) make()` | Identity (no planes on or off); cached singleton |
| `static CPT(RenderAttrib) make_all_off()` | Turns off all inherited planes; cached singleton |
| `static CPT(RenderAttrib) make_default()` | Same as `make()` |
| `int get_num_on_planes() const` | |
| `NodePath get_on_plane(int n) const` | Sorted by pointer order |
| `bool has_on_plane(const NodePath &plane) const` | |
| `int get_num_off_planes() const` | |
| `NodePath get_off_plane(int n) const` | |
| `bool has_off_plane(const NodePath &plane) const` | Also true if `has_all_off()` and not explicitly on |
| `bool has_all_off() const` | |
| `bool is_identity() const` | No on/off planes, not all-off |
| `CPT(RenderAttrib) add_on_plane(const NodePath &plane) const` | |
| `CPT(RenderAttrib) remove_on_plane(const NodePath &plane) const` | |
| `CPT(RenderAttrib) add_off_plane(const NodePath &plane) const` | |
| `CPT(RenderAttrib) remove_off_plane(const NodePath &plane) const` | |
| `CPT(ClipPlaneAttrib) filter_to_max(int max_clip_planes) const` | Keep highest-priority `n` on-planes; memoized |

## Usage
```cpp
NodePath plane_np = render.attach_new_node(new PlaneNode("clip", LPlane(0, 0, 1, 0)));
node_path.set_attrib(DCAST(ClipPlaneAttrib, ClipPlaneAttrib::make())->add_on_plane(plane_np));
```

## See also
[RenderAttrib](RenderAttrib.md), [PlaneNode](PlaneNode.md),
[LightAttrib](LightAttrib.md) (same add/remove-on/off-list pattern),
[pgraph README — the state pipeline](README.md#the-state-pipeline)
