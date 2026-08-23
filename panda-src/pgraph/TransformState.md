# TransformState

**Source:** `panda/src/pgraph/transformState.h` (+ `.I`, `.cxx`; NOT
`transformState_ext.h/.cxx`, Python glue)
**Inherits:** NodeCachedReferenceCount

Represents a node's coordinate-system transform — the position/rotation/
scale counterpart to [RenderState](RenderState.md), managed with the exact
same immutable/interned/ref-counted pattern (never constructed or mutated
directly; always via `make_*()`, and "changing" it means composing to get a
new interned instance). Every `PandaNode` carries one; `NodePath::set_pos()`
etc. all resolve to swapping the node's `TransformState`.

## Behavior notes

- **Dual representation, lazily reconciled.** A `TransformState` can be
  built componentwise (`make_pos_hpr_scale_shear()`, or any of the
  `make_pos`/`make_hpr`/`make_quat`/`make_scale`/`make_shear` convenience
  forms) *or* from an arbitrary matrix (`make_mat()`). Whichever form it
  was built from is what `components_given()`/`hpr_given()`/`quat_given()`
  report; the other representation is computed lazily and cached on first
  access (`check_mat()`/`check_components()`/`check_hpr()`/`check_quat()`
  each gate a `do_calc_*()`). A general matrix built via `make_mat()` that
  happens to decompose losslessly into pos/hpr/scale/shear will still
  report `components_given() == false` — it remembers how it was made, not
  just what it's equivalent to.
- **Identity and invalid are both special-cased singletons**
  (`_identity_state`/`_invalid_state`), checked first in `compose()`/
  `invert_compose()`/`compare_to()`/`operator==` as fast paths — "all
  invalid transforms are equivalent to each other, and all identity
  transforms are equivalent to each other" regardless of how they'd
  otherwise compare.
- **Invalid propagates.** `compose()`/`invert_compose()`: if either side is
  invalid, the result is invalid (returns whichever side was invalid,
  without composing). A `make_mat()`/`make_pos_hpr_scale_shear()` etc. call
  given NaN components returns `make_invalid()` instead of asserting in
  release builds (`nassertr(..., make_invalid())`).
- **Singular matrices produce invalid, not garbage, inverses**:
  `invert_compose()` computes `this`'s inverse lazily and caches it in
  `_inv_mat`; if the matrix turns out singular (`is_singular()`, itself
  lazily computed by attempting the invert), the result is
  `make_invalid()` rather than a divide-by-zero/NaN matrix. Same for
  componentwise inversion when `scale == 0`.
- **Componentwise fast path**: `do_compose()`/`do_invert_compose()` use a
  cheap quaternion+scale composition instead of full 4x4 matrix multiply
  when both sides have uniform scale and componentwise data
  (`compose_componentwise` config var gates this, on by default) —
  matrix-based composition is the fallback for general/sheared/non-uniform
  transforms.
- Composition results are cached per-pair exactly like `RenderState`
  (`_composition_cache`/`_invert_composition_cache`, gated by the
  `transform-cache` config var), with the same paired-cache-entry cleanup
  scheme on destruction.
- `get_uniform_scale()` asserts (in debug) that `has_uniform_scale()` is
  true — calling it on a non-uniformly-scaled transform is a usage error,
  not something that silently returns one axis.
- `is_2d()` tracks whether the transform only ever touched X/Y (built via
  the `*2d()` factory methods) — 2D nodes (e.g. some `pgui`/`text` UI trees)
  use this to stay in the cheaper 2D transform path.

## API

| Method | Notes |
|---|---|
| `static CPT(TransformState) make_identity()` / `make_invalid()` | Canonical singletons |
| `static CPT(TransformState) make_pos/hpr/quat/scale/shear(...)` | Single-component factories |
| `static CPT(TransformState) make_pos_hpr_scale(...)` / `make_pos_quat_scale(...)` | Common combined factories |
| `static CPT(TransformState) make_pos_hpr_scale_shear(...)` / `make_pos_quat_scale_shear(...)` | Full componentwise factories |
| `static CPT(TransformState) make_mat(const LMatrix4&)` | From an arbitrary matrix |
| `static CPT(TransformState) make_pos2d/rotate2d/scale2d/shear2d(...)` etc. | 2D equivalents |
| `CPT(TransformState) compose(other) const` / `invert_compose(other) const` | Cached composition |
| `CPT(TransformState) set_pos/hpr/quat/scale/shear(...) const` | Return a new state with one component replaced |
| `CPT(TransformState) get_inverse() const` | This transform's inverse as a TransformState |
| `bool is_identity() const` / `is_invalid() const` / `is_singular() const` / `is_2d() const` | State queries |
| `bool has_pos/hpr/quat/scale/shear/mat() const` | Is a given representation currently known/cached? |
| `bool has_uniform_scale() const` / `has_identity_scale() const` / `has_nonzero_shear() const` | Cheap-path detection |
| `const LPoint3 &get_pos() const` / `get_hpr()` / `get_quat()` / `get_norm_quat()` / `get_scale()` / `get_uniform_scale()` / `get_shear()` / `get_mat()` | Accessors (lazily computed if needed) |
| `int compare_to(other) const` / `bool operator==(other) const` | Ordering/equality for interning |
| `static int garbage_collect()` / `clear_cache()` | Same caching maintenance as RenderState |

## Usage

```cpp
CPT(TransformState) t = TransformState::make_pos_hpr_scale(
    LPoint3(0, 10, 0), LVecBase3(90, 0, 0), LVecBase3(1, 1, 1));
node_path.set_transform(t);

CPT(TransformState) relative = a.get_transform()->invert_compose(b.get_transform());
```

## See also

[RenderState](RenderState.md), [PandaNode](PandaNode.md),
[NodePath](NodePath.md), module [README](README.md) ("The state pipeline").
