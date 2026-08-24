# TypedWritable / TypedWritableReferenceCount

**Source:** `panda/src/putil/typedWritable.h` / `.I` / `.cxx`, `typedWritableReferenceCount.h` / `.I` / `.cxx`
**Inherits:** `TypedWritable : TypedObject`; `TypedWritableReferenceCount : TypedWritable, ReferenceCount`
**Inherited by:** almost everything serializable in Panda — nodes, geoms, textures, animation data, etc.

Base classes for anything that can be written to and read from a **Bam
file** (Panda's binary serialization format — see [BamReader](BamReader.md)
/ [BamWriter](BamWriter.md)). `bam.h` is just an umbrella include that pulls
in version constants (`_bam_major_ver` etc.) and the Bam config variables;
it has no API of its own.

`TypedWritable` alone is *not* reference-counted — most real classes need
both typed-writable-ness and ref-counting, so `TypedWritableReferenceCount`
exists purely to avoid every such class having to multiply-inherit from
`TypedWritable` and `ReferenceCount` itself.

## Behavior notes

- **The four-method lifecycle is entirely opt-in via no-op defaults.**
  `write_datagram()`, `fillin()`, `complete_pointers()`, and `finalize()` all
  default to doing nothing (or returning 0/false); a subclass only overrides
  what it actually needs to serialize. See [BamWriter](BamWriter.md) /
  [BamReader](BamReader.md) for the full write/read protocol these hook into.
- **`complete_pointers()` exists because of forward references.** When
  `fillin()` calls `BamReader::read_pointer()` for a pointer to an object
  that hasn't been read yet, no pointer is available immediately — instead
  the call is queued, and later the `BamReader` calls back into
  `complete_pointers(TypedWritable **p_list, BamReader*)` with one resolved
  pointer per `read_pointer()` call made during `fillin()`, in the same
  order. The override is responsible for counting its own `read_pointer()`
  calls and pulling exactly that many entries back out of `p_list`.
  `require_fully_complete()` (default `false`) lets a class demand that
  *its own* nested pointers be fully resolved before its own
  `complete_pointers()` runs — but the class must then be careful about
  circular references, since that combination makes the object formally
  unreadable from a self-referential bam stream.
- **`finalize()` is a separate, later callback** for setup that needs *all*
  objects in the file (not just this one's pointers) to be resolved; a class
  requests it by calling `BamReader::register_finalize(this)` during
  `fillin()`/`complete_pointers()`.
- **`get_bam_modified()`/`mark_bam_modified()` drive live-update
  re-transmission**, not disk serialization — a `BamWriter` tracks the last
  `UpdateSeq` it wrote for an object, and `consider_update()`/
  `write_pointer()` re-write the object if `mark_bam_modified()` was called
  since. Ordinary file-based bam writing doesn't need this; it matters for
  streaming a live scene graph to a connected client.
- **`update_bam_nested()`** is a separate hook (default no-op) called when
  the object *itself* is unmodified but a `BamWriter` doing a live-update
  pass still needs to recurse into children that might be dirty — see
  `BamWriter::consider_update()`.
- **Objects track which `BamWriter`s reference them, via a lock-free
  singly-linked list (`_bam_writers`)** so the destructor can call
  `BamWriter::object_destructs()` on each one — this lets a still-open
  `BamWriter` release its `object_id` and know to write a "freed" record
  (`BOC_remove`) into the stream, without the `TypedWritable` needing a
  back-pointer to just one writer. The list head pointer doubles as a
  spinlock (its low bit is a "locked" flag), so `add_bam_writer()` /
  `remove_bam_writer()` are safe to call from any thread.
- **`encode_to_bam_stream()`/`decode_raw_from_bam_stream()` are
  single-object convenience wrappers** around a full `BamWriter`/`BamReader`
  pair backed by an in-memory `DatagramBuffer` — handy for serializing one
  object to `bytes` without setting up a `.bam` file, but less efficient
  than reusing one `BamWriter`/`BamReader` if you have many objects.
  `decode_raw_from_bam_stream()` can only recover objects that are also
  `ReferenceCount`-derived (it needs to `ref()` the result to keep it alive
  once the local `BamReader` goes away) — for non-refcounted `TypedWritable`s
  you must drive a `BamReader` directly.
- **`TypedWritableReferenceCount::decode_from_bam_stream()`** is the
  friendlier variant of the above for classes that *do* derive from both
  bases — it returns one `PT(TypedWritableReferenceCount)` directly instead
  of separate typed/refcounted pointers.
- **`PointerToBase<TypedWritableReferenceCount>::update_type()` is
  specialized to a no-op** in `typedWritableReferenceCount.h` — a
  memory-usage-tracking hook that doesn't apply here since the type is
  already known statically.
- Both classes register an **alternate type name** with a typo
  (`"TypedWriteable"` / `"TypedWriteableReferenceCount"`) for backward
  compatibility reading old bam files that used the misspelling.

## API

### TypedWritable
| Signature | Notes |
|---|---|
| `virtual void write_datagram(BamWriter*, Datagram&)` | Serialize this object's data; default no-op |
| `virtual void fillin(DatagramIterator&, BamReader*)` | Deserialize (called by generated `make_from_bam()` and for live updates); default no-op |
| `virtual int complete_pointers(TypedWritable **p_list, BamReader*)` | Resolve forward-referenced pointers queued during `fillin()`; returns count consumed from `p_list`; default 0 |
| `virtual bool require_fully_complete() const` | If true, nested pointers must be complete before this object's `complete_pointers()` runs; default `false` |
| `virtual void finalize(BamReader*)` | Post-all-objects-resolved hook; only called if `register_finalize()` was requested; default no-op |
| `virtual void update_bam_nested(BamWriter*)` | Recurse into children during a live-update write pass; default no-op |
| `virtual ReferenceCount *as_reference_count()` | `nullptr` unless overridden (by `TypedWritableReferenceCount`) |
| `void mark_bam_modified()` / `UpdateSeq get_bam_modified() const` | Bump/read the modification counter used for live-update re-transmission |
| `vector_uchar encode_to_bam_stream() const` / `bool encode_to_bam_stream(vector_uchar&, BamWriter* = nullptr) const` | Serialize just this object to a byte buffer |
| `static bool decode_raw_from_bam_stream(TypedWritable*&, ReferenceCount*&, vector_uchar data, BamReader* = nullptr)` | Inverse of the above; only works if the object is also `ReferenceCount`-derived |

### TypedWritableReferenceCount
| Signature | Notes |
|---|---|
| `static PT(TypedWritableReferenceCount) decode_from_bam_stream(vector_uchar data, BamReader* = nullptr)` | Friendlier decode for classes inheriting both bases |
| `virtual ReferenceCount *as_reference_count()` | Returns `this` |

## Usage

```cpp
class MyThing : public TypedWritableReferenceCount {
public:
  void write_datagram(BamWriter *manager, Datagram &dg) override {
    TypedWritableReferenceCount::write_datagram(manager, dg);
    dg.add_int32(_value);
    manager->write_pointer(dg, _child);   // may be nullptr
  }

  void fillin(DatagramIterator &scan, BamReader *manager) override {
    TypedWritableReferenceCount::fillin(scan, manager);
    _value = scan.get_int32();
    manager->read_pointer(scan);          // resolved later
  }

  int complete_pointers(TypedWritable **p_list, BamReader *manager) override {
    int index = TypedWritableReferenceCount::complete_pointers(p_list, manager);
    _child = DCAST(MyThing, p_list[index++]);
    return index;
  }

private:
  int _value = 0;
  PT(MyThing) _child;
};
```

## See also

[BamReader.md](BamReader.md) (drives `fillin()`/`complete_pointers()`/`finalize()`) ·
[BamWriter.md](BamWriter.md) (drives `write_datagram()`/`update_bam_nested()`) ·
[Factory.md](Factory.md) (creates instances by `TypeHandle` for the reader) ·
[README.md](README.md)
