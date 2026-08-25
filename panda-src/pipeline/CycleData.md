# CycleData

**Source:** `panda/src/pipeline/cycleData.{h,I,cxx}`
**Inherits from:** `NodeReferenceCount` (if `DO_PIPELINING`) else `MemoryBase`

Base class for "a single page of data maintained by a
[`PipelineCycler`](PipelineCycler.md)." To protect some data across pipeline
stages, subclass `CycleData` (conventionally as a nested `CData` class) to
hold that data's fields, and give the owning class a
`PipelineCycler<CData> _cycler` member.

## Behavior

Which base class `CycleData` inherits depends on `DO_PIPELINING`: with it
defined, a containing class stores a *pointer* to `CycleData` per stage (via
the cycler), so it must be reference-counted — specifically `NodeReferenceCount`
rather than plain `ReferenceCount`, "since we want to make a distinction
between references within the cycler, and references outside the cycler
(e.g. `GeomPipelineReader`)". That distinction is exactly what
[`PipelineCyclerTrueImpl::write_stage()`](PipelineCycler.md#behavior) checks
via `get_node_ref_count()` to decide whether a copy-on-write is needed.
Without `DO_PIPELINING`, the containing class stores the `CycleData` object
directly (no pointer, no cycler indirection), so it derives from the
non-reference-counted `MemoryBase` instead.

`make_copy()` is pure virtual — every subclass must implement it to return a
new heap copy of itself; this is what copy-on-write calls. The `write_datagram`/
`fillin`/`complete_pointers` overloads are no-ops by default, present so
subclasses that need Bam (Panda's binary serialization format) persistence
for their cycled data can override them. `get_parent_type()` defaults to
`TypeHandle::none()` and exists purely for debugging/`output()` — "returns
the type of the container that owns the CycleData."

## API reference

```cpp
class CycleData : public NodeReferenceCount /* or MemoryBase */ {
public:
  CycleData() = default;
  CycleData(const CycleData &copy) = default;
  virtual ~CycleData();

  virtual CycleData *make_copy() const = 0;

  virtual void write_datagram(BamWriter *, Datagram &) const;
  virtual void write_datagram(BamWriter *, Datagram &, void *extra_data) const;
  virtual int complete_pointers(TypedWritable **p_list, BamReader *manager);
  virtual void fillin(DatagramIterator &scan, BamReader *manager);
  virtual void fillin(DatagramIterator &scan, BamReader *manager, void *extra_data);

  virtual TypeHandle get_parent_type() const;
  virtual void output(std::ostream &out) const;
};
```

## Usage

Not used standalone — subclass it (typically named `CData`, nested inside the
class it protects), implement `make_copy()`, and wire it up to a
`PipelineCycler<CData>` member. Access the current data through
[`CycleDataReader`](CycleDataReader.md)/[`CycleDataLockedReader`](CycleDataLockedReader.md)/
[`CycleDataWriter`](CycleDataWriter.md), never by touching the cycler or the
`CycleData` pointer directly.

## Related classes

- [`PipelineCycler`](PipelineCycler.md) — owns the per-stage `CycleData`
  pointers and performs the copy-on-write
- [`CycleDataReader`](CycleDataReader.md), [`CycleDataLockedReader`](CycleDataLockedReader.md),
  [`CycleDataWriter`](CycleDataWriter.md) — the accessor classes
