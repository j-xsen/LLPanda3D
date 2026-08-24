# DataNodeTransmit

**Source:** `panda/src/dgraph/dataNodeTransmit.h` / `.I` / `.cxx`
**Inherits:** `TypedWritable`
**Used by:** [DataNode](DataNode.md) (as its input/output payload), [DataGraphTraverser](DataGraphTraverser.md) (passes these between nodes)

`DataNodeTransmit` is the data container passed into and out of
`DataNode::transmit_data()`/`do_transmit_data()` — one instance holds the
values for every input (or output) wire of a single node, indexed by the
`int` returned from `define_input()`/`define_output()`. Internally it's
just `pvector<EventParameter>`, sized on demand.

## Behavior notes

- **Slots are lazily created, sparse-safe, and capped at 1000.**
  `set_data(index, ...)` grows the internal vector on demand via
  `slot_data()`, which `nassertv(index < 1000)`s — writing to an index of
  1000+ is a programming error caught by assertion, not a graceful failure.
  There's no corresponding cap check on `get_data()`/`has_data()` since they
  only read within the current size.
- **Reading an unset or out-of-range index returns a shared empty
  `EventParameter` by reference**, not a null pointer or exception —
  `get_data()` returns a reference to a local `static EventParameter
  empty_parameter`. Safe because the data graph is single-threaded (see
  [DataNode](DataNode.md#behavior-notes)); do not rely on this for
  cross-thread reads.
- **`has_data()` checks `!is_empty()`, not just "index in range."** An index
  within bounds that was never `set_data()`'d, or was explicitly set to a
  default-constructed `EventParameter`, reports `has_data() == false`.
- **Bam (de)serialization only round-trips `TypedWritableReferenceCount`-held
  values.** `write_datagram()` writes each stored `EventParameter` via
  `param.get_ptr()` through `BamWriter::write_pointer()` — an `EventParameter`
  wrapping a plain value (int/double/string via `EventStoreInt` etc. still
  works, since those *are* `TypedWritableReferenceCount` subclasses) survives
  a save/load round trip, but the parameter's *identity as a specific index*
  is not re-derived from anything the node itself declared; `fillin()` just
  re-reads the same count of pointers into a flat vector — the caller must
  already know the wire layout to make sense of it.

## API

| Signature | Notes |
|---|---|
| `DataNodeTransmit()` | Empty; zero slots |
| `void reserve(int num_wires)` | Pre-sizes the internal vector — callers typically pass `get_num_inputs()`/`get_num_outputs()` |
| `const EventParameter &get_data(int index) const` | Returns the shared empty parameter if `index` is unset or out of range |
| `bool has_data(int index) const` | False for unset, out-of-range, or explicitly-empty slots |
| `void set_data(int index, const EventParameter &data)` | Grows the vector as needed (up to index 999) |

## Usage

```cpp
// Inside DataNode::do_transmit_data() overrides:
void MyNode::
do_transmit_data(DataGraphTraverser *, const DataNodeTransmit &input,
                  DataNodeTransmit &output) {
  if (input.has_data(_matrix_input)) {
    const EventParameter &p = input.get_data(_matrix_input);
    // ... read p ...
  }
  output.set_data(_result_output, EventParameter(some_value));
}
```

## See also

[README.md](README.md) for the shared data-graph model ·
[DataNode](DataNode.md) (declares the wires this indexes into) ·
[DataGraphTraverser](DataGraphTraverser.md) (moves these between nodes each frame)
