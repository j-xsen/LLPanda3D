# OmniBoundingVolume

**Source:** `panda/src/mathutil/omniBoundingVolume.h` (+ `.I`, `.cxx`)
**Inherits:** `GeometricBoundingVolume` (→ [BoundingVolume.md](BoundingVolume.md))
**Inherited by:** (none)

A bounding volume that always fills all of space — every query against it
answers "yes, completely." It's the concrete class that actually implements
the `is_infinite()` state as its *only* state (there's no way to make an
`OmniBoundingVolume` non-infinite); used as the bounds for things that should
never be culled (e.g. explicitly disabling frustum culling on a node by
giving it an `OmniBoundingVolume`).

## Behavior notes

- **Every method is a trivial constant-answer override — there is no
  internal state at all.** `contains_*()` always returns
  `IF_possible|IF_some|IF_all`; `extend_by_*()`/`around_*()` always return
  `true` having done nothing (extending/surrounding an omni volume is a
  no-op — it's already everything); `extend_other()`/`around_other()`
  instead mutate the *other* volume, calling `other->set_infinite()` — i.e.
  `extend_by(omni)` on some other volume correctly makes *that* volume
  infinite, which is the one place an `OmniBoundingVolume` actually affects
  something outside itself.
- **`xform()` is a no-op** (transforming "everything" leaves it as
  "everything") and **`get_approx_center()` always returns the origin** —
  there's no meaningful center for an infinite volume, so it just picks a
  fixed, cheap answer rather than asserting.
- **Construction never fails and there's no empty state** — unlike every
  other bounding volume in this module, an `OmniBoundingVolume` has no
  `is_empty()` path; it's unconditionally infinite from construction.

## API

| Signature | Notes |
|---|---|
| `OmniBoundingVolume()` | Always infinite; no other constructor |
| `virtual LPoint3 get_approx_center() const` | Always the origin |
| `virtual void xform(const LMatrix4 &mat)` | No-op |
| `virtual void output(std::ostream&) const` | `"omni"` |

## Usage

```cpp
// Disable frustum culling for this node's subtree:
some_node_path.node()->set_bounds(new OmniBoundingVolume());
some_node_path.node()->set_final(true);
```

## See also

[BoundingVolume.md](BoundingVolume.md) · [CompositeBoundingVolumes.md](CompositeBoundingVolumes.md) ·
[README.md](README.md)
