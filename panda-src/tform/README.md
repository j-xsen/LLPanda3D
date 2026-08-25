# TFORM — Panda3D's Mouse/Keyboard Input & Transform Devices

**Source:** `panda/src/tform/` · Library: `libp3tform` · Notify category:
`tform`

`tform` supplies the concrete [data graph](../dgraph/README.md) node classes
that turn raw mouse/keyboard input into either UI events (button-down,
region enter/leave, IME candidates) or scene-graph transforms (orbit the
scene with a mouse, drive around on a horizontal plane). Where
[dgraph](../dgraph/README.md) is just the traversal plumbing, `tform` is the
actual content flowing through it — parented below a device node like
`MouseAndKeyboard` (see [display](../display/README.md)), and typically
above whatever consumes its outputs (a camera `NodePath`, an
`EventHandler`).

This directory documents the public C++ API of every class in
`panda/src/tform`.

## Class map

```
DataNode
├── MouseInterfaceNode                (MouseInterfaceNode.md) — button-gated
│   │                                                            base for input TFormers
│   ├── DriveInterface                (DriveInterface.md)    — horizontal-plane
│   │                                                           "drive a vehicle" transform
│   ├── Trackball                     (Trackball.md)         — Performer-style
│   │                                                           orbit/pan/dolly transform
│   └── MouseSubregion                (MouseSubregion.md)    — rescales mouse
│                                                                input from a screen sub-rect
├── ButtonThrower                     (ButtonThrower.md)     — converts button
│                                                                events into Panda Events
├── MouseWatcher (also MouseWatcherBase)
│                                     (MouseWatcher.md)      — region-based mouse/
│                                                                keyboard UI dispatch
└── Transform2SG                      (Transform2SG.md)      — applies a transform
                                                                 input to a scene-graph node

(no common base)
├── MouseWatcherBase                  (MouseWatcherBase.md)  — sorted region
│   │                                                           collection, shared impl
│   └── MouseWatcherGroup             (MouseWatcherGroup.md) — ref-counted
│                                                                standalone region group
├── MouseWatcherRegion                (MouseWatcherRegion.md) — one named
│                                                                 rectangular hit-test region
└── MouseWatcherParameter             (MouseWatcherParameter.md) — event
                                                                     argument passed to region callbacks
```

`config_tform.h`/`.cxx` holds only `tform`-category config variables
(`drive-forward-speed`, `trackball-use-alt-keys`, `inactivity-timeout`,
etc.) and type-registration boilerplate — no class of its own.

## Core concepts

**Every TFormer is just a `DataNode` — the transform/event semantics are
entirely a convention of which wires it defines and consumes.** There's no
special "TFormer" base class or interface; the term (used loosely in
several of these classes' doc comments) just names the pattern of a
`DataNode` that reads mouse/button wires and emits a `transform`
([TransformState](../pgraph/TransformState.md)) wire for something further
down — typically a [Transform2SG](Transform2SG.md) — to apply.
[MouseInterfaceNode](MouseInterfaceNode.md) formalizes only the
button-gating half of this convention (`require_button()`/`watch_button()`),
shared by [DriveInterface](DriveInterface.md), [Trackball](Trackball.md),
and [MouseSubregion](MouseSubregion.md).

**Two independent input models coexist in this directory: raw button
events and region-based hit-testing.** [ButtonThrower](ButtonThrower.md)
works purely on the `button_events` wire, with no notion of screen position
— every button becomes an `Event` (or is passed through), regardless of
where the mouse is. [MouseWatcher](MouseWatcher.md), by contrast, tests
mouse position against a collection of
[MouseWatcherRegion](MouseWatcherRegion.md)s and only fires
[MouseWatcherParameter](MouseWatcherParameter.md)-carrying callbacks/events
for buttons pressed over (or, for keyboard-flagged regions, anywhere while)
a region is relevant. Both can be present in the same data graph
simultaneously, watching the same underlying `button_events` wire, since
neither consumes/removes events the other needs — though a
[MouseWatcherRegion](MouseWatcherRegion.md)'s `SF_other_button`/
`SF_mouse_button` suppress flags can stop [MouseWatcher](MouseWatcher.md)
from forwarding events to whatever's parented below *it*.

**"Entered," "within," "preferred," and "over" are four related but
distinct ideas**, all defined by [MouseWatcher](MouseWatcher.md) and
[MouseWatcherRegion](MouseWatcherRegion.md) together — see
[MouseWatcher](MouseWatcher.md)'s behavior notes for the exact
enter-vs-within dispatch logic and how `_enter_multiple` collapses the
distinction between them.

**Config-variable defaults (from `config_tform.h`) seed constructor state,
they are not read live.** [DriveInterface](DriveInterface.md)'s speed/dead-
zone/ramp-time fields, and [Trackball](Trackball.md)'s
`trackball_use_alt_keys`-gated modifier watching, are all copied from the
`ConfigVariableDouble`/`ConfigVariableBool` globals once, in the
constructor — changing the config variable after a node is constructed has
no effect on that already-constructed instance.

## See also

[dgraph](../dgraph/README.md) (the data graph traversal machinery every
class here builds on) · [display/MouseAndKeyboard](../display/MouseAndKeyboard.md)
(typical upstream source of raw `xy`/`pixel_xy`/`button_events`) ·
[pgraph/TransformState](../pgraph/TransformState.md) (the value type carried
on every `transform` wire in this directory)
