# UnionBoundingVolume / IntersectionBoundingVolume

**Source:** `panda/src/mathutil/unionBoundingVolume.h` (+ `.I`, `.cxx`) ·
`intersectionBoundingVolume.h` (+ `.I`, `.cxx`)
**Inherits:** `GeometricBoundingVolume` (→ [BoundingVolume.md](BoundingVolume.md)), both
**Inherited by:** (none)

CSG-style combinators: a `UnionBoundingVolume` contains a point iff *any* of
its components does; an `IntersectionBoundingVolume` contains a point iff
*all* of its components do. Both hold an arbitrary list of
`GeometricBoundingVolume` components (`pvector<CPT(GeometricBoundingVolume)>`)
and are themselves `GeometricBoundingVolume`s, so they can nest (a union of
unions, an intersection containing a union, etc.).

## Behavior notes

- **Their default (empty-component) states are opposite, and it's load-bearing.**
  A `UnionBoundingVolume` with no components starts/resets to `F_empty`
  (union of nothing = nothing) — see its constructor and `clear_components()`.
  An `IntersectionBoundingVolume` with no components starts/resets to
  `F_infinite` (intersection of nothing = no constraint = everything) — see
  its `clear_components()`. Getting these backwards would silently invert
  every empty-composite query.
- **`add_component()` on both classes does redundancy pruning as components
  are added, not just a plain list append.** `UnionBoundingVolume::add_component()`:
  if the new component is fully contained in an existing one (`existing->contains(component)
  & IF_all`), it's dropped as redundant; if an existing component is fully
  contained in the new one, the existing one is dropped instead. If the new
  component itself `is_infinite()`, the whole union collapses to infinite
  and every other component is discarded (`_components.clear()`).
  `IntersectionBoundingVolume::add_component()` does the mirror-image
  pruning, plus two extra special cases:
  - Adding a `UnionBoundingVolume` component first calls
    `filter_intersection()` on a *copy* of that union, discarding any of the
    union's own sub-components that can't possibly intersect this
    intersection's existing components (`UnionBoundingVolume::filter_intersection()`,
    which drops components where `volume->contains(existing) & IF_possible`
    is false) — an optimization to avoid keeping useless union branches. If
    that leaves only one sub-component, the union collapses to just that
    component.
  - Adding another `IntersectionBoundingVolume` component flattens it —
    each of *its* components is added individually via recursive
    `add_component()` calls, rather than nesting one intersection inside
    another.
  - If any pairwise `contains()` between the new component and an existing
    one comes back exactly `0` (`IF_no_intersection`), the whole
    intersection immediately collapses to empty (`_flags = F_empty`) and
    all components are cleared — an intersection with any two mutually
    exclusive components can never contain anything.
- **`get_approx_center()` on `UnionBoundingVolume` is the *average* of all
  component centers** (not the true centroid weighted by volume, and not
  the true geometric center of the union shape) — a cheap approximation
  matching the "approx" in the method name. `IntersectionBoundingVolume`'s
  version follows the same averaging pattern (see its `.cxx` if exact
  behavior for a specific pairing matters).
- **`xform()` on both replaces each component with a freshly-transformed
  copy** (`component->make_copy()` then `xform()` on the copy) rather than
  mutating shared component pointers in place — necessary since components
  are held as `CPT` (const pointer) and may be shared with other composite
  volumes.
- **`contains_point()`/`contains_lineseg()` fold results across all
  components with early-exit**: `UnionBoundingVolume` ORs each component's
  result together and can stop early once `IF_all` is achieved (further
  components can't add more certainty); `IntersectionBoundingVolume` ANDs
  results together and stops early once the running result loses
  `IF_possible` (once *no* intersection is possible, more components can't
  revive it). Either encountering `IF_dont_understand` from a component
  short-circuits the whole query to `IF_dont_understand`.
- **`contains_union()`/`contains_intersection()` (double-dispatch pairing
  when one composite is tested against another) delegate to
  `other_contains_union()`/`other_contains_intersection()`** — a private
  helper on the composite class itself, rather than being implemented on
  the generic `BoundingVolume` base, since only the composite knows how to
  walk its own component list against an opaque "other" volume.
- Both allocate via `ALLOC_DELETED_CHAIN`.

## API

### Shared shape
| Signature | Notes |
|---|---|
| `int get_num_components() const` / `const GeometricBoundingVolume *get_component(int n) const` | `MAKE_SEQ components` |
| `void clear_components()` | Resets to the type's identity state (empty for Union, infinite for Intersection) |
| `void add_component(const GeometricBoundingVolume *component)` | Prunes redundant components; see behavior notes |
| `virtual LPoint3 get_approx_center() const` | Average of component centers |
| `virtual void xform(const LMatrix4 &mat)` | Replaces each component with a transformed copy |
| `virtual void output(std::ostream&) const` / `write(std::ostream&, int) const` | `"union [ ... ]"` / `"intersection { ... }"` |

### UnionBoundingVolume extra
| Signature | Notes |
|---|---|
| `void filter_intersection(const BoundingVolume *volume)` | Drops components that can't possibly intersect `volume` — used internally by `IntersectionBoundingVolume::add_component()` |

## Usage

```cpp
PT(UnionBoundingVolume) region = new UnionBoundingVolume();
region->add_component(sphere_a);
region->add_component(sphere_b);   // dropped automatically if fully inside sphere_a

PT(IntersectionBoundingVolume) overlap = new IntersectionBoundingVolume();
overlap->add_component(frustum_bounds);
overlap->add_component(region);    // collapses to empty if provably disjoint
```

## See also

[BoundingVolume.md](BoundingVolume.md) · [OmniBoundingVolume.md](OmniBoundingVolume.md) ·
[README.md](README.md)
