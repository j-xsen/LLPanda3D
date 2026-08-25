# DataNode

**Source:** `panda/src/dgraph/dataNode.h` / `.I` / `.cxx`
**Inherits:** `PandaNode`
**Inherited by:** device/transform classes in [tform](../tform/README.md) — `MouseWatcher`, `DriveInterface`, `Trackball`, `ButtonThrower`, etc.

`DataNode` is the abstract base for every node in the [data graph](README.md).
By itself it defines no inputs or outputs — subclasses call `define_input()`/
`define_output()` in their constructor to declare named, typed wires, then
override `do_transmit_data()` to read inputs and produce outputs each frame.

## Behavior notes

- **Copying a `DataNode` does not copy its inputs, outputs, or connections.**
  The copy constructor (`dataNode.I`) only copies the base `PandaNode`; a
  copied node starts with zero wires and must have its subclass constructor
  re-run (or otherwise redefine wires) to be usable — `make_copy()` on plain
  `DataNode` returns a genuinely empty node.
- **Wires can't be redefined once connections exist.** `define_input()` and
  `define_output()` both `nassertr(_data_connections.empty(), 0)` — safe to
  call in a constructor (before any parenting), but calling either after the
  node has already been wired into the graph is a programming error caught
  by assertion, not silently handled.
- **Connections are automatic and name+type based, not declared per-edge.**
  `reconnect()` runs from `parents_changed()` — i.e. any time this node's
  parent set changes via the scene-graph API — and rebuilds
  `_data_connections` by matching each input wire's name against every
  `DataNode` parent's output wires. A same-named wire with a mismatched
  `TypeHandle` is skipped with a `dgraph_cat.warning()`, not connected. If
  *multiple* `DataNode` parents expose an output matching the same input
  name, every match is still connected — `transmit_data()` applies them in
  parent order and overwrites, so the highest-indexed matching parent wins,
  silently; `reconnect()` only logs a debug message in this case, not a
  warning.
- **A node with inputs defined, at least one `DataNode` parent, but zero
  resulting connections logs a warning** (`"No data connected to ..."`) —
  useful signal that a name/type mismatch exists somewhere upstream.
- **`transmit_data()` (called by [DataGraphTraverser](DataGraphTraverser.md))
  is not the method to override** — it handles collecting matched parent
  outputs into a fresh input `DataNodeTransmit` and (in a debug build with
  `dgraph_cat` at `spam` level) logs every named input/output value. Override
  `do_transmit_data()` instead, which receives the already-assembled inputs.
- **Wire index numbers are assignment order, not stable across redefinition.**
  `define_input()`/`define_output()` return an `int` index (0, 1, 2, ...in
  first-defined order) that subclasses must cache and use to index into the
  `DataNodeTransmit` passed to `do_transmit_data()` — re-defining an
  already-named wire keeps its original index and just changes its type.

## API

### Construction
| Signature | Notes |
|---|---|
| `explicit DataNode(const std::string &name)` | `name` is the node name, same as `PandaNode` |

### Wires (call from a subclass constructor)
| Signature | Notes |
|---|---|
| `int define_input(const std::string &name, TypeHandle data_type)` | Declares/updates an input wire; returns its stable index into `do_transmit_data()`'s `input` |
| `int define_output(const std::string &name, TypeHandle data_type)` | Declares/updates an output wire; returns its stable index into `do_transmit_data()`'s `output` |
| `int get_num_inputs() const` / `int get_num_outputs() const` | Wire counts — sizes the `DataNodeTransmit` a caller should prepare |

### Transmission
| Signature | Notes |
|---|---|
| `void transmit_data(DataGraphTraverser *trav, const DataNodeTransmit inputs[], DataNodeTransmit &output)` | Called by `DataGraphTraverser`; assembles matched parent data, then calls `do_transmit_data()` |
| `virtual void do_transmit_data(DataGraphTraverser *trav, const DataNodeTransmit &input, DataNodeTransmit &output)` | **Override this in subclasses.** Base implementation is a no-op |

### Introspection
| Signature | Notes |
|---|---|
| `void write_inputs(std::ostream &out) const` / `write_outputs(...)` | Dumps declared wire names + types |
| `void write_connections(std::ostream &out) const` | Dumps each active connection: input wire name/type and the parent it's connected from |

## Usage

```cpp
class MyDevice : public DataNode {
public:
  explicit MyDevice(const std::string &name) : DataNode(name) {
    _matrix_output = define_output("matrix", LMatrix4::get_class_type());
  }

protected:
  void do_transmit_data(DataGraphTraverser *, const DataNodeTransmit &,
                         DataNodeTransmit &output) override {
    output.set_data(_matrix_output, EventParameter(compute_matrix()));
  }

private:
  int _matrix_output;
};

// Wiring: ordinary NodePath reparenting under a data-graph root.
NodePath data_root("data_root");
NodePath device_np = data_root.attach_new_node(new MyDevice("dev"));
DataGraphTraverser trav;
trav.traverse(data_root.node());  // runs every frame
```

## See also

[README.md](README.md) for the shared data-graph model ·
[DataNodeTransmit](DataNodeTransmit.md) (the per-node data container) ·
[DataGraphTraverser](DataGraphTraverser.md) (drives `transmit_data()` calls)
