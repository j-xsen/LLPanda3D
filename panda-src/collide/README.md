# collide — Panda3D's C++ Collision Detection System

**Source:** `panda/src/collide/` · Library: `libp3collide` · Notify category: `collide`

A separate, lightweight collision system independent of the `bullet`/`physx`
integrations — pure CPU shape-vs-shape testing built directly on the scene
graph (`pgraph`).

It has three layers:

1. **Solids** — `CollisionSolid` is the abstract shape base (sphere, box,
   capsule, plane, polygon, ray, line, segment, parabola, floor mesh).
   Attached to the scene graph via `CollisionNode`.
2. **Traversal** — `CollisionTraverser` walks a scene graph subtree each
   frame, finds every pair of solids whose bounding volumes overlap, and
   double-dispatches the real per-shape intersection test, producing a
   `CollisionEntry` for each hit.
3. **Handling** — `CollisionHandler` is the abstract "what to do with a hit"
   interface: `CollisionHandlerEvent` throws named events, `*Queue` just
   remembers entries, and the `*Physical` family (`Pusher`, `FluidPusher`,
   `Floor`, `Gravity`) actually move the colliding NodePath in response.

## Class map

```
CopyOnWriteObject
└── CollisionSolid                     (CollisionSolid.md)     — abstract shape base
    ├── CollisionSphere                (CollisionSphere.md)
    │   └── CollisionInvSphere         (CollisionInvSphere.md) — inside-out sphere
    ├── CollisionBox                   (CollisionBox.md)
    ├── CollisionCapsule               (CollisionCapsule.md)   — aka "CollisionTube" (deprecated alias)
    ├── CollisionPlane                 (CollisionPlane.md)     — infinite half-space
    │   └── CollisionPolygon           (CollisionPolygon.md)   — convex, planar, finite
    │       └── CollisionGeom          (internal — see CollisionPolygon.md "See also")
    ├── CollisionRay                   (CollisionRay.md)       — infinite one direction
    │   └── CollisionLine              (CollisionLine.md)      — infinite both directions
    ├── CollisionSegment               (CollisionSegment.md)   — finite, no radius
    ├── CollisionParabola              (CollisionParabola.md)  — finite arc
    └── CollisionFloorMesh             (CollisionFloorMesh.md) — triangle soup, ray-only

PandaNode
└── CollisionNode                      (CollisionNode.md)      — holds CollisionSolids in the scene graph

TypedWritableReferenceCount
└── CollisionEntry                     (CollisionEntry.md)     — one detected collision

Namable
└── CollisionTraverser                 (CollisionTraverser.md) — drives the traversal + double-dispatch

TypedReferenceCount
└── CollisionHandler                   (CollisionHandler.md)   — abstract "what happens on a hit"
    ├── CollisionHandlerQueue          (CollisionHandlerQueue.md)  — just remembers entries
    └── CollisionHandlerEvent          (CollisionHandlerEvent.md)  — throws named events
        ├── CollisionHandlerHighestEvent (CollisionHandlerHighestEvent.md) — only the closest hit
        └── CollisionHandlerPhysical   (CollisionHandlerPhysical.md) — abstract, moves the collider
            ├── CollisionHandlerPusher (CollisionHandlerPusher.md)
            │   └── CollisionHandlerFluidPusher (CollisionHandlerFluidPusher.md)
            ├── CollisionHandlerFloor  (CollisionHandlerFloor.md)
            └── CollisionHandlerGravity (CollisionHandlerGravity.md)

TypedObject (debug-only, DO_COLLISION_RECORDING)
└── CollisionRecorder                  (CollisionRecorder.md)
    └── CollisionVisualizer : PandaNode, CollisionRecorder (CollisionVisualizer.md)
```

Not documented as standalone files here (out of scope / internal-only):
- **`config_collide.h/.cxx`** — module config; variables summarized below.
- **`collisionTube.h`** — not a class, just `typedef CollisionCapsule
  CollisionTube;` kept for backward compatibility. See
  [CollisionCapsule.md](CollisionCapsule.md).
- **`collisionGeom.h`** (`CollisionGeom`) — a `CollisionPolygon` subclass
  created on-the-fly by `CollisionTraverser` when colliding against a
  `Geom`'s raw triangles (not the pre-built solids in a `CollisionNode`).
  Never construct one yourself. See "See also" in
  [CollisionPolygon.md](CollisionPolygon.md).
- **`collisionLevelStateBase.h`/`collisionLevelState.h`** — private
  per-recursion-level bookkeeping (`CollisionLevelStateBase`, template
  `CollisionLevelState<MaskType>`) that `CollisionTraverser` uses internally
  during its recursive scene graph walk; not part of the public API. Summarized
  under "How traversal works" in [CollisionTraverser.md](CollisionTraverser.md).
- **`collisionTraverser_ext.h`, `collisionHandlerEvent_ext.h`,
  `collisionHandlerQueue_ext.h`, `collisionHandlerPhysical_ext.h`** — Python
  `__getstate__`/`__reduce__`/`__setstate__` pickle-support glue
  (`Extension<T>` specializations), not real C++ API surface.

## Core concepts

**Every intersection test is `from`-shape-vs-`into`-shape, and it's not
symmetric.** A `CollisionSolid` only implements the "from" side of the tests
it needs to actually initiate (e.g. `CollisionRay` implements
`test_intersection()` but most shapes don't need to *be* a ray-from-shape
test target beyond the default no-op). `CollisionSolid::test_intersection()`
dispatches by the *into* solid's runtime type to one of
`test_intersection_from_sphere/box/line/ray/segment/capsule/parabola()`,
which the *from* solid overrides for the pairs it supports. An unhandled
combination calls `report_undefined_intersection_test()`, which logs an
error via the `collide` notify category once and returns no collision
(never crashes) — so an exotic from/into pairing (e.g. "from a plane") just
silently detects nothing rather than throwing.

**Collide masks gate *which* solids are even tested against each other, as a
cheap pre-filter before the real geometric test.** Every `CollisionNode` has
a `from_collide_mask` (what its solids can detect) and an `into_collide_mask`
(what can detect them, inherited from `PandaNode`); a pair is only tested if
`(from.get_from_collide_mask() & into.get_into_collide_mask()) != 0`. By
convention (`panda/src/putil/collideMask.h`): the low 20 bits are reserved
for `CollisionNode`s (`default_collision_node_collide_mask =
CollideMask::lower_on(20)`, set as *both* from- and into-mask on a new
`CollisionNode`), bit 20 is the default for ordinary `GeomNode`s
(`default_geom_node_collide_mask`), and bits 21-31 are unassigned — custom
collider/target pairs are typically scoped using dedicated bits above 20. `CollisionNode`
additionally restricts itself to `get_legal_collide_mask() ==
CollideMask::all_on()` (no restriction), unlike renderable geometry nodes.

**"Tangible" controls physical response, not detection.** Every
`CollisionSolid` has an `is_tangible()` flag (default `true`). An intangible
solid still generates `CollisionEntry`s and still fires
`CollisionHandlerEvent` patterns — it's just invisible to the `*Physical`
handlers' actual movement logic (`CollisionHandlerPusher` won't push against
it, etc.). This is the standard way to build a "trigger volume" that reacts
to being entered/exited without blocking movement.

**`CollisionTraverser::traverse()` recurses the scene graph exactly once per
call, testing all registered colliders in parallel as it goes** (not once
per collider) — see [CollisionTraverser.md](CollisionTraverser.md) for the
`CollisionLevelState` bookkeeping and the `collider_sort` ordering knob.
`add_collider(NodePath, CollisionHandler*)` associates one `CollisionNode`
NodePath with the handler that should process its hits; a `CollisionNode`
can only be driven by one handler at a time (calling `add_collider()` again
on the same NodePath just retargets it), but one handler can serve many
colliders.

**`respect_prev_transform` enables swept/continuous collision for fast
movers.** When true (`CollisionTraverser::set_respect_prev_transform()` or
the global `respect-prev-transform` Config.prc variable, default off), a
`CollisionRay`/`CollisionSegment`-style "from" solid effectively also
considers where the collider *was* last frame, catching thin-wall
tunneling that a single-frame point-in-time test would miss.

## Config variables (`config_collide.cxx`)

| Variable | Default | Affects |
|---|---|---|
| `respect-prev-transform` | `false` | `CollisionTraverser::get_respect_prev_transform()` default — see swept-collision note above |
| `respect-effective-normal` | `true` | Whether `CollisionSolid::get_effective_normal()` overrides a shape's real geometric normal in intersection results (used by `CollisionPolygon`/floor-type solids to report a flattened "up" normal regardless of actual facet slope) |
| `allow-collider-multiple` | `false` | Lets `CollisionTraverser::traverse()` walk the scene graph once per distinct collider bitmask width (single/double/quad) instead of always using the widest, when more than one collider is registered |
| `flatten-collision-nodes` | `true` | Whether the scene graph flattener is allowed to combine sibling `CollisionNode`s |
| `collision-parabola-bounds-threshold` / `collision-parabola-bounds-sample` | `1.0` / `10` | Tuning for `CollisionParabola`'s bounding-volume approximation (a parabola has no exact analytic bounding sphere, so it's sampled) |
| `fluid-cap-amount` | `1` | Caps how many extra collision-timing subdivisions `CollisionHandlerFluidPusher` will do per frame |
| `pushers-horizontal` | `false` | Default value of `CollisionHandlerPusher::get_horizontal()` for newly-constructed pushers |

## File index

| Topic | Purpose |
|---|---|
| [CollisionSolid.md](CollisionSolid.md) | Abstract shape base: tangible/effective-normal flags, double-dispatch test machinery |
| [CollisionNode.md](CollisionNode.md) | `PandaNode` holding a list of `CollisionSolid`s; collide masks |
| [CollisionEntry.md](CollisionEntry.md) | One detected collision: surface point/normal, contact point/normal, `t` |
| [CollisionTraverser.md](CollisionTraverser.md) | Drives the scene-graph walk, owns colliders + handlers |
| [CollisionHandler.md](CollisionHandler.md) | Abstract "what to do with a hit" interface |
| [CollisionHandlerEvent.md](CollisionHandlerEvent.md) | Throws named events (`%fn`/`%in`/`%ig`/`%ft`/... pattern substitution) |
| [CollisionHandlerHighestEvent.md](CollisionHandlerHighestEvent.md) | Like Event, but only the single closest hit per group |
| [CollisionHandlerQueue.md](CollisionHandlerQueue.md) | Just remembers entries for the caller to poll (picking) |
| [CollisionHandlerPhysical.md](CollisionHandlerPhysical.md) | Abstract base for handlers that reposition the collider's target NodePath |
| [CollisionHandlerPusher.md](CollisionHandlerPusher.md) | Shoves the target out of solid geometry |
| [CollisionHandlerFluidPusher.md](CollisionHandlerFluidPusher.md) | `Pusher` variant that uses collision timing to avoid tunneling |
| [CollisionHandlerFloor.md](CollisionHandlerFloor.md) | Snaps target Z to the highest ray hit each frame |
| [CollisionHandlerGravity.md](CollisionHandlerGravity.md) | Floor-following + actual gravity/velocity integration |
| [CollisionSphere.md](CollisionSphere.md) | Spherical solid |
| [CollisionInvSphere.md](CollisionInvSphere.md) | Inside-out sphere (solid outside, empty inside) |
| [CollisionBox.md](CollisionBox.md) | Axis-aligned-at-construction cuboid solid |
| [CollisionCapsule.md](CollisionCapsule.md) | Cylinder with hemispherical endcaps (formerly misnamed "CollisionTube") |
| [CollisionPlane.md](CollisionPlane.md) | Infinite half-space |
| [CollisionPolygon.md](CollisionPolygon.md) | Convex, planar, finite polygon |
| [CollisionFloorMesh.md](CollisionFloorMesh.md) | Triangle mesh tested only against vertical rays |
| [CollisionRay.md](CollisionRay.md) | Infinite ray (one direction), no radius |
| [CollisionLine.md](CollisionLine.md) | Infinite line (both directions) |
| [CollisionSegment.md](CollisionSegment.md) | Finite line segment, no radius |
| [CollisionParabola.md](CollisionParabola.md) | Finite arc along a parabola (projectile paths) |
| [CollisionRecorder.md](CollisionRecorder.md) | Debug-only base for recording every test the traverser makes |
| [CollisionVisualizer.md](CollisionVisualizer.md) | Debug-only `PandaNode` that renders tested/detected polygons live |

## Status

collide — done (2026-08-23). See [../../README.md](../../README.md) for the
overall index across `panda/src/*` modules.
