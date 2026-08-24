# PGUI — Panda3D's C++ GUI Toolkit

**Source:** `panda/src/pgui/` · Library: `libp3pgui` · Notify category: `pgui`

PGUI is Panda3D's built-in immediate-scenegraph GUI system: buttons, text
entries, sliders, scroll frames, wait bars. It is a thin layer over the regular
3-D scene graph plus the `MouseWatcher` input system — every widget is a
`PandaNode` with a `MouseWatcherRegion` attached, and interaction is delivered
as Panda events (`throw_event`) with the string-based naming convention
`<prefix><id>`, e.g. `press-mouse1-my_button`. DirectGUI (Python) is a wrapper
around exactly these classes; there is no separate C++/Python widget set.

This directory documents the public C++ API of every class in `panda/src/pgui`,
for use without re-reading the engine source.

## Class map

```
PandaNode
└── PGItem                    (PGItem.md)   — base class for every widget
    ├── PGButton               (PGButton.md)
    ├── PGEntry                (PGEntry.md)
    ├── PGSliderBar             (PGSliderBar.md)
    ├── PGWaitBar               (PGWaitBar.md)
    └── PGVirtualFrame          (PGVirtualFrame.md)
        └── PGScrollFrame       (PGScrollFrame.md)
PGTop                          (PGTop.md)    — root node, not a PGItem
CullTraverser
└── PGCullTraverser            (PGCullTraverser.md)  — internal, rarely touched directly

PGFrameStyle                   (PGFrameStyle.md)     — standalone value type, not a node

MouseWatcherRegion
├── PGMouseWatcherRegion       (PGMouseWatcherRegion.md)  — one per PGItem, internal
└── PGMouseWatcherBackground   (PGMouseWatcherBackground.md)
MouseWatcherGroup
└── PGMouseWatcherGroup        (PGMouseWatcherGroup.md)   — one per PGTop, internal

TypedWritableReferenceCount, MouseWatcherParameter
└── PGMouseWatcherParameter    (PGMouseWatcherParameter.md) — event payload type
```

`PGButtonNotify`, `PGItemNotify`, and `PGSliderBarNotify` (the C++ callback
interfaces) are documented as subsections inside the docs of the class they
belong to (`PGItem.md`, `PGButton.md`, `PGSliderBar.md`) rather than as
standalone files — they're small, pure-virtual mixins with no independent use.

## Core concepts (shared by every widget)

**Everything must live under a `PGTop`.** `PGTop` (see [PGTop.md](PGTop.md)) is
the root of a PG hierarchy; it must be parented into the 2-D scene graph
(typically `aspect2d`) and given a `MouseWatcher` via `set_mouse_watcher()`.
Every `PGItem` must be parented somewhere below a `PGTop`, or it will render
but never receive mouse/keyboard events. Panda's default `aspect2d` /
`base.mouseWatcherNode` setup in `ShowBase` already does this; a scene graph
built manually in pure C++ must wire this up explicitly.

**Frame vs. state defs.** Every `PGItem` has:
- a **frame** (`set_frame(left, right, bottom, top)`): the clickable rectangle,
  in the item's local coordinate space (X = right, Z = up, in the `rfu`
  convention used throughout this code — a 2-D frame's 4 components are
  `(left, right, bottom, top)`, mapped to X and Z).
- one or more **state defs** (`get_state_def(n)` → `NodePath`): separate render
  subgraphs, one of which is displayed at a time according to `set_state(n)`.
  Concrete widgets define their own meaning for state numbers (see each
  class's `State` enum) and switch between them automatically in response to
  mouse events (e.g. `PGButton` cycles ready/depressed/rollover/inactive).
- a **frame style** per state (`set_frame_style(state, PGFrameStyle)`): the
  procedurally-generated background/border geometry for that state — see
  [PGFrameStyle.md](PGFrameStyle.md).

**Events, not virtual overrides, are the primary extension point.** `PGItem`
defines virtual hook methods (`press()`, `release()`, `enter_region()`, ...)
that concrete widgets override to update their own state, but the intended way
for *application* code to react to a widget is to listen for the thrown Panda
event. Every event name is `get_..._prefix() + get_id()` (or
`+ button.get_name() + "-" + get_id()` for button-keyed events), where
`get_id()` defaults to an internal auto-generated name but can be overridden
with `set_id()`. See the "Events" section of [PGItem.md](PGItem.md) for the
full list of base events (enter/exit/within/without/focus/press/repeat/release/
keystroke), and each widget's own doc for its additional events (`PGButton`'s
`click-`, `PGEntry`'s `accept-`/`type-`/`erase-`/..., `PGSliderBar`'s
`adjust-`).

**The C++ Notify interfaces are the alternative, non-event extension point.**
Instead of (or in addition to) listening for string events, C++ code can
subclass `PGItemNotify` (or the more specific `PGButtonNotify` /
`PGSliderBarNotify`) and call `item->set_notify(this)` to get direct virtual
callbacks — no string formatting/parsing, no global event dispatch. Panda uses
this internally (`PGScrollFrame` watches its own slider bars this way,
`PGSliderBar` watches its own thumb/left/right buttons this way). A notify
object does **not** own the items that point to it, and vice versa; on
destruction each side automatically detaches (see [PGItem.md](PGItem.md)
"Notify interface").

**Locking.** Every `PGItem` (and subclass) has an internal `LightReMutex _lock`
acquired by nearly every method. The classes are therefore safe to touch from
multiple threads doing independent scene-graph work, but this is an
implementation detail, not an invitation to fine-grained cross-thread
choreography.

**Everything is a `PandaNode`.** Standard `NodePath` operations
(`reparent_to`, `set_pos`, `set_scale`, `hide`/`show`, ...) work on any PGUI
widget exactly as on any other node. `xform()` is overridden on the PG classes
to keep the frame/frame-style/axis in sync when the node is scaled or moved
directly through the scene graph.

## File index

| Class | Purpose |
|---|---|
| [PGItem.md](PGItem.md) | Base class: frame, state defs, events, notify, focus, sounds |
| [PGFrameStyle.md](PGFrameStyle.md) | Procedural border/background geometry descriptor |
| [PGTop.md](PGTop.md) | Root node of a PG hierarchy; owns the `MouseWatcher` link |
| [PGButton.md](PGButton.md) | Clickable button with ready/depressed/rollover/inactive states |
| [PGEntry.md](PGEntry.md) | Single/multi-line editable text field |
| [PGSliderBar.md](PGSliderBar.md) | Draggable slider / scroll bar, with optional thumb + end buttons |
| [PGWaitBar.md](PGWaitBar.md) | Simple progress/loading bar |
| [PGVirtualFrame.md](PGVirtualFrame.md) | Scissor-clipped window onto an oversized "virtual canvas" child |
| [PGScrollFrame.md](PGScrollFrame.md) | `PGVirtualFrame` + automatic scroll bars (the "scrolled panel" widget) |
| [PGCullTraverser.md](PGCullTraverser.md) | Internal cull traverser that registers regions during rendering |
| [PGMouseWatcherRegion.md](PGMouseWatcherRegion.md) | Internal: the `MouseWatcherRegion` owned by each `PGItem` |
| [PGMouseWatcherGroup.md](PGMouseWatcherGroup.md) | Internal: the region group owned by each `PGTop` |
| [PGMouseWatcherBackground.md](PGMouseWatcherBackground.md) | Internal: routes keypresses to background-focus items |
| [PGMouseWatcherParameter.md](PGMouseWatcherParameter.md) | Event payload type attached to PG input events |

## Status

pgui — done (2026-08-22). Other `panda/src/*` subsystems not yet documented —
see `../../README.md` for the overall index.
