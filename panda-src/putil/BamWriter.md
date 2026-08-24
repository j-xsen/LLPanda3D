# BamWriter

**Source:** `panda/src/putil/bamWriter.h` / `.cxx`
**Inherits:** `BamEnums` (enums documented in [BamReader.md](BamReader.md#bamenums-defined-here-shared-by-bamwriter))

The write-side counterpart of [BamReader](BamReader.md) — serializes
[TypedWritable](TypedWritable.md) objects (and everything they transitively
reference) to a `DatagramSink`. Handles object-graph deduplication (each
object written exactly once, referenced thereafter by ID), forward
management of the queue of not-yet-written referenced objects, and
(optionally) live re-transmission of objects that change after their first
write.

## Behavior notes

- **`write_object()` is breadth-queued, not naively recursive.** It assigns
  the root object an ID via `enqueue_object()`, then `flush_queue()` drains
  a FIFO of queued objects — each object's `write_datagram()` may call
  `write_pointer()` on other objects, which enqueues *them* too, so the
  queue keeps growing until everything reachable has been written. The very
  first datagram of a `write_object()` call is tagged `BOC_push`; the whole
  call is closed with a trailing `BOC_pop` datagram (see
  [`BamObjectCode`](BamReader.md#bamenums-defined-here-shared-by-bamwriter)),
  bracketing one root write as a nesting level for the reader.
- **`write_pointer()` writes only an object ID inline**; if the target
  hasn't been written yet (or has been modified since — see below), it
  transparently enqueues the object for writing in this same
  `write_object()`/`flush_queue()` pass rather than writing it out
  immediately at the point of reference. This is the mechanism that lets
  `BamReader::read_pointer()` receive a forward reference.
- **Two independent staleness tracks exist per object, both keyed by
  `UpdateSeq`**: `_written_seq` (which top-level `write_object()`/
  `consider_update()` pass last wrote this object — used to avoid
  re-visiting an object twice within one pass) and `_modified` (the
  object's own `get_bam_modified()` value as of the last write — used to
  detect the object changed *since* it was last written, for live-update
  streaming). A stale object is transparently re-queued and rewritten with
  a fresh `TypeHandle`+data record instead of a pointer to what's already on
  the stream.
- **`consider_update()`/`update_bam_nested()` are the live-update path.**
  `TypedWritable::update_bam_nested()` (default no-op) should call
  `BamWriter::consider_update()` on each of its own children so a
  long-lived `BamWriter` streaming a mutable scene graph to a client can
  recursively find and push only what actually changed, without re-walking
  and re-writing everything.
- **TypeHandles are interned into the stream exactly like `BamReader`
  expects**: `write_handle()` writes an index (the `TypeHandle`'s own
  internal index number, reused as-is — "why not?"), and only the *first*
  occurrence of a given index is followed by the type's name and recursive
  parent-type list. Before writing, the type is snapped to the nearest
  ancestor the reader will actually be able to construct
  (`BamReader::get_factory()->find_registered_type()`), with a warning if
  even that fails.
- **Object/PTA IDs escalate 16-bit → 32-bit at the same threshold as the
  reader**: once an assigned ID reaches `0xffff`, `_long_object_id` (or
  `_long_pta_id`) flips permanently and all subsequent IDs of that kind are
  written as `uint32`. This must stay symmetric with `BamReader`'s read-side
  escalation.
- **Obsolete type-name history is a static, `BamWriter`-instance-independent
  table** (`obsolete_type_names`, a `SimpleHashMap` chosen specifically "to
  avoid static init ordering issues"). `record_obsolete_type_name()` lets a
  class register that it used to be serialized under an older name before
  some bam version, so `get_obsolete_type_name()` can pick the
  version-appropriate name when `set_file_minor_ver()`/`bam-version` targets
  an older format. Registering also adds the old name as a *read-side*
  alternate name via `TypeRegistry`, so old files remain loadable.
- **`bam-version` config variable can force writing an older minor
  version** (down to 6.21 — earlier isn't supported at all); `init()`
  refuses to proceed and clamps to the nearest supported bound (with an
  error) if the configured version is out of range.
- **The destructor releases writer-held references, not the objects
  themselves** — while an object is queued/pending in `_state_map` but not
  yet written, `BamWriter` holds a `ref()` on it (if it's `ReferenceCount`-
  derived) so it can't be deleted out from under the write; that ref is
  released as soon as the object is actually written (`flush_queue()`), or
  by the destructor for anything still pending when the writer itself goes
  away.
- **`object_destructs()` is invoked by `TypedWritable`'s own destructor**
  (they're mutual friends), not polled — when a written object is deleted,
  its ID is queued into `_freed_object_ids` and flushed as a `BOC_remove`
  record before the next `write_object()` call, so the reader can release
  its own bookkeeping for that ID.

## API

| Signature | Notes |
|---|---|
| `explicit BamWriter(DatagramSink *target = nullptr)` | |
| `void set_target(DatagramSink*)` / `get_target()` | Also calls `init()` implicitly if needed; flushes the old target first |
| `bool init()` | Writes the file header (version, endianness); honors `bam-version` config |
| `bool write_object(const TypedWritable *obj)` | Writes one root object plus everything it transitively references |
| `bool has_object(const TypedWritable *obj) const` | Whether this object has been written (or queued) at all |
| `void flush()` | Flushes the underlying `DatagramSink` |
| `void consider_update(const TypedWritable *obj)` | Live-update entry point — see behavior notes |
| `void write_pointer(Datagram&, const TypedWritable *dest)` | The pointer-writing half of the read/write pointer protocol |
| `void write_file_data(SubfileInfo &result, const Filename&)` / `(..., const SubfileInfo &source)` | Emits a large out-of-band data block (paired with `read_file_data()`) |
| `void write_cdata(Datagram&, const PipelineCyclerBase&)` / `(..., void *extra_data)` | Writes a pipelined `CycleData` via its `write_datagram()` |
| `bool register_pta(Datagram&, const void *ptr)` | PTA dedup — returns `true` if already written (see `WRITE_PTA()` macro) |
| `void write_handle(Datagram&, TypeHandle)` | Writes an interned `TypeHandle` |
| `static std::string get_obsolete_type_name(TypeHandle, int major, int minor)` | |
| `static void record_obsolete_type_name(TypeHandle, std::string name, int before_major, int before_minor)` | |
| `int get_file_major_ver() const` / `get_file_minor_ver() const` / `set_file_minor_ver(int)` | |
| `BamEndian get_file_endian() const` / `bool get_file_stdfloat_double() const` | |
| `BamTextureMode get_file_texture_mode() const` / `set_file_texture_mode(BamTextureMode)` | |
| `TypedWritable *get_root_node() const` / `set_root_node(TypedWritable*)` | Scene-graph root, for correctly writing `NodePath`s |

## Usage

```cpp
BamWriter writer(&sink);
if (!writer.init()) { /* ... */ }
writer.write_object(root);   // recursively writes root + everything it references
writer.flush();
```

## See also

[BamReader.md](BamReader.md) (the matching read side — see its `BamEnums`
section) · [TypedWritable.md](TypedWritable.md) (`write_datagram()`/
`update_bam_nested()` contract) · [BamCache.md](BamCache.md) ·
[README.md](README.md)
