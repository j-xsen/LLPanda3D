# CollisionVisualizer

**Source:** `panda/src/collide/collisionVisualizer.h` (+ `.I`, `.cxx`)
**Inherits:** `PandaNode`, [CollisionRecorder](CollisionRecorder.md)

Shows the polygons that are detected as collisions, as well as those that
are merely considered for collisions. It may be parented anywhere in the
scene graph, where it renders directly. The concrete debug tool behind
[CollisionTraverser](CollisionTraverser.md)`::show_collisions()`; only
compiled in when `DO_COLLISION_RECORDING` is defined.

## Behavior notes

- **Usually created indirectly, via `CollisionTraverser::show_collisions(root)`**
  rather than constructed and wired up by hand — that call builds one,
  parents it under `root`, and installs it as the traverser's recorder in
  one step; `hide_collisions()` reverses it.
- **Draws detected-vs-tested-but-missed solids differently** — internally
  tracks per-`CollisionSolid`, per-transform detected/missed counts
  (`SolidInfo`) plus a set of surface/contact points/normals, and renders
  them with distinguishable coloring/wireframe, making it possible to
  distinguish what actually collided from what was merely bounds-checked
  and rejected.
- **`point_scale`/`normal_scale` control the on-screen size of the drawn
  point markers and normal-vector lines** — adjustable when the default
  visualization is too small or too large relative to the scene's scale.
- **Multiple inheritance from `PandaNode` and `CollisionRecorder` (both
  ultimately `TypedObject`) requires the explicit `as_typed_object()`
  disambiguation accessor** — needed anywhere code needs a plain
  `TypedObject*` from a `CollisionVisualizer*` without the compiler getting
  confused about which base's `TypedObject` is meant.
- **`clear()` resets all accumulated per-frame recording state** without
  detaching the node from the scene graph.

## API

| Signature | Notes |
|---|---|
| `explicit CollisionVisualizer(const std::string &name)` | |
| `void set_point_scale(PN_stdfloat)` / `PN_stdfloat get_point_scale() const` | Size of drawn hit-point markers |
| `void set_normal_scale(PN_stdfloat)` / `PN_stdfloat get_normal_scale() const` | Length of drawn normal lines |
| `void clear()` | Resets accumulated recording state |

## Usage

```cpp
// typically driven through the traverser, not constructed directly:
CollisionVisualizer *viz = ctrav.show_collisions(render);
viz->set_point_scale(0.5);
// later:
ctrav.hide_collisions();
```

## See also

[CollisionRecorder.md](CollisionRecorder.md) · [CollisionTraverser.md](CollisionTraverser.md)
· [README.md](README.md)
