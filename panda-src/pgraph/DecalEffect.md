# DecalEffect

**Source:** `panda/src/pgraph/decalEffect.h` (+ `.I`, `.cxx`)
**Inherits:** RenderEffect

Applied to a `GeomNode` to indicate that its children are coplanar and
should be drawn as decals — eliminating Z-fighting between geometry drawn
at (nearly) the same depth, like a bullet-hole decal on a wall.

## Behavior notes

- Pure marker effect: carries no data (`compare_to_impl()` always returns
  0 — all `DecalEffect` instances are equivalent) and `make()` takes no
  arguments.
- `safe_to_combine()` returns `false` — a decal-effect node must not be
  merged with sibling nodes during flattening, since the decal relationship
  depends on this specific node's children/ordering.
- The actual depth-fighting fix (rendering the decal children with a small
  depth or polygon offset instead of true coplanar depth) is implemented
  downstream in the cull/GSG layer, not in this class — see the
  `depth-offset-decals` config variable (module README) which selects
  between a depth-offset strategy and a polygon-offset strategy for
  rendering the effect.

## API

| Method | Notes |
|---|---|
| `static CPT(RenderEffect) make()` | No parameters — marker effect |

## Usage

```cpp
// geom_node is a GeomNode whose children are coplanar decal geometry.
NodePath(geom_node).set_effect(DecalEffect::make());
```

## See also

- [RenderEffect](RenderEffect.md) — base class
- [../pgraph/README.md](README.md) — `depth-offset-decals` config variable
