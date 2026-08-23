# ParamNodePath

**Source:** `panda/src/pgraph/paramNodePath.h` (+ `.I`, `.cxx`)
**Inherits:** ParamValueBase

A thin `ParamValueBase` wrapper that lets a [NodePath](NodePath.md) be
stored and passed around as a generic, type-erased parameter value (e.g.
for shader inputs or anywhere Panda's generic-parameter system is used).
Almost entirely boilerplate — construct one from a `NodePath`, read it back
with `get_value()`.

## Behavior notes

- Bam I/O has a version fork: files older than bam minor version 40 could
  not serialize a `NodePath` directly, so `write_datagram()`/`fillin()`
  fall back to writing/reading just the bare `PandaNode` pointer for those
  files (losing path-disambiguation information) instead of the full
  `NodePath`.

## API

| Method | Notes |
|---|---|
| `ParamNodePath(const NodePath&)` / `(NodePath&&)` | Wrap a NodePath (copy or move) |
| `get_value() const` | Returns the wrapped `NodePath` |
| `get_value_type() const` | Returns `NodePath::get_class_type()` |

## Usage

```cpp
PT(ParamNodePath) param = new ParamNodePath(some_node_path);
NodePath np = param->get_value();
```

## See also

[NodePath](NodePath.md)
