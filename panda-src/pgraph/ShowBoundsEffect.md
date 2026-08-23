# ShowBoundsEffect

**Source:** `panda/src/pgraph/showBoundsEffect.h` (+ `.I`, `.cxx`)
**Inherits:** RenderEffect

Applied to a `GeomNode` to cause a visible bounding volume to be drawn for
it — a development/debugging aid for tracking down bounding-volume issues,
equivalent to `NodePath::show_bounds()`.

## Behavior notes

- Single flag, `tight`: false (default) draws the node's existing cached
  bounding volume as-is (cheap); true draws a *tight* bounding box
  recomputed from the node's actual vertices every frame (more accurate,
  more expensive — the source comment ties this directly to
  `NodePath::show_tight_bounds()`).
- `safe_to_combine()` returns `false` — must not be merged with sibling
  nodes during flattening, since the debug-draw is specific to this node's
  own bounds.
- Purely a debug marker; the actual bounds-drawing logic lives downstream
  in the cull/draw pipeline (not in this class).

## API

| Method | Notes |
|---|---|
| `static CPT(RenderEffect) make(bool tight = false)` | |
| `bool get_tight() const` | |

## Usage

```cpp
// Equivalent to node_path.show_bounds():
node_path.set_effect(ShowBoundsEffect::make());

// Equivalent to node_path.show_tight_bounds():
node_path.set_effect(ShowBoundsEffect::make(true));
```

## See also

- [RenderEffect](RenderEffect.md) — base class
- [../pgraph/README.md](README.md) — module overview
