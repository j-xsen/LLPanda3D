# DGRAPH — Panda3D's Data Graph

**Source:** `panda/src/dgraph/` · Library: `libp3dgraph` · Notify category: `dgraph`

The data graph is a second, parallel node hierarchy — built from the same
`PandaNode`/`NodePath` machinery as the 3-D scene graph, but used to wire up
input devices (mice, trackballs, joysticks) to the objects that consume their
data (a `Trackball` computing a matrix, a node that applies that matrix to
the scene graph) each frame. See [tform](../tform/README.md) for the
concrete device/consumer classes; this module is just the plumbing: the node
type and the traversal that drives it.

This directory documents the public C++ API of every class in
`panda/src/dgraph`.

## Class map

```
PandaNode
└── DataNode                     (DataNode.md)   — abstract base; defines
                                                     named typed inputs/outputs

TypedWritable
└── DataNodeTransmit              (DataNodeTransmit.md)  — one node's worth
                                                             of input or output data

DataGraphTraverser                (DataGraphTraverser.md) — stack-allocated,
                                                              drives one traversal
```

## Core concepts

**A data graph is a `PandaNode` tree, traversed top-down once per frame.**
Each `DataNode` declares named, typed *input wires* and *output wires* in its
constructor (via `define_input()`/`define_output()`). When a `DataNode` is
parented under another `DataNode`, it automatically connects any input whose
name+type matches one of its new parent's outputs — no explicit wiring calls
needed, just `NodePath` reparenting. A `DataGraphTraverser` walks the tree,
calling `transmit_data()` on each `DataNode` and passing its `DataNodeTransmit`
output down to matched children's inputs.

**Connections are name+type matched, not declared explicitly, and are
runtime-only.** `DataNode::reconnect()` runs automatically from
`parents_changed()` (i.e. whenever a `DataNode`'s parent set changes) and
rebuilds `_data_connections` from scratch each time by scanning all
`DataNode` parents for an output wire matching each input wire's *name*.
Mismatched types for a same-named wire are silently skipped with a
`dgraph_cat.warning()`; if more than one parent's output matches the same
input name, a debug message logs it, but *every* match is still recorded as
a connection — at transmit time they're applied in parent order, so the
highest-indexed matching parent silently wins (last write to that input
slot). These connections are never serialized to a
Bam file — only the base `PandaNode` state is (`DataNode::write_datagram`/
`fillin` just delegate to `PandaNode`) — so after loading from Bam, wiring
depends on `parents_changed()` firing again to rebuild `_data_connections`.

**Multi-parent nodes wait for every parent before transmitting.**
`DataGraphTraverser::traverse_below()` transmits immediately into a
single-parent child, but a child with N `DataNode` parents is held in
`_multipass_data` until data has arrived via all N paths, then transmitted
once. If the traversal never reaches all paths (e.g. one parent lies outside
the data-graph root passed to `traverse()`), `collect_leftovers()` forces a
transmit anyway at the end and logs a warning
(`"improperly parented partly outside of data graph"`).

**Non-`DataNode` nodes in the middle of the hierarchy don't block
propagation.** `traverse_below()` recurses through any `PandaNode` child that
isn't itself a `DataNode`, passing the same `DataNodeTransmit` output further
down — so a `DataNode` several levels below a plain `PandaNode` (or below a
device node that was itself just an anchor point) can still receive data,
as long as name/type matching still lines up when a `DataNode` descendant is
finally reached.

**Single-threaded by design.** `DataNode` "does not attempt to cycle its data
with a `PipelineCycler`" (see `dataNode.h`) — the data graph is intended to
be traversed by exactly one thread (normally the App thread), unlike the
scene graph.

## See also

[tform](../tform/README.md) — the concrete device and transform classes
(`MouseWatcher`, `DriveInterface`, `Trackball`, `ButtonThrower`, ...) that
subclass `DataNode` and give the data graph its actual content.
