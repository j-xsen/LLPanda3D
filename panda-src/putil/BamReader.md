# BamReader

**Source:** `panda/src/putil/bamReader.h` / `.I` / `.cxx` (+ `bamReaderParam.h` / `.I` / `.cxx`,
`bamEnums.h` / `.cxx`)
**Inherits:** `BamEnums`
**Related:** `BamReaderAuxData : TypedReferenceCount` (nested-friend helper, same header)

The fundamental interface for extracting binary [TypedWritable](TypedWritable.md)
objects from a Bam file — the abstract counterpart of [BamWriter](BamWriter.md).
It reads from a `DatagramGenerator` (not necessarily a disk file), reconstructs
objects via the global [Factory\<TypedWritable\>](Factory.md), and stitches
inter-object pointers back together across a possibly-forward-referencing
stream. `.bam` files are scene graphs by convention; a Bam file may contain
any list of `TypedWritable` objects, conventionally named `.boo` in that more
general usage. `BamFile` (not in this module) is the higher-level "just read
a `.bam` off disk" wrapper around this class.

## BamEnums (defined here, shared by BamWriter)

```cpp
enum BamEndian { BE_bigendian = 0, BE_littleendian = 1, BE_native = <platform> };
enum BamObjectCode { BOC_push, BOC_pop, BOC_adjunct, BOC_remove, BOC_file_data };
enum BamTextureMode { BTM_unchanged, BTM_fullpath, BTM_relative, BTM_basename, BTM_rawdata };
```
- `BamEndian` — declared byte order of numeric data in the file (each object
  may or may not respect it).
- `BamObjectCode` — precedes every top-level datagram in bam format ≥6.21:
  `BOC_push`/`BOC_pop` bracket a nesting level, `BOC_adjunct` is a plain
  object at the current level, `BOC_remove` introduces a list of object IDs
  the writer will no longer reference (safe to free), `BOC_file_data` marks
  an out-of-band large-data block that a following object will claim via
  `read_file_data()`.
- `BamTextureMode` — how a `BamWriter` should record texture image paths
  (unchanged / absolute / relative / basename-only / embed raw pixels).
- Each has `operator<<`/`operator>>` for config-file/log formatting, defined
  in `bamEnums.cxx`.

## Behavior notes

- **Reading one logical object can consume many datagrams.** `read_object()`
  reads the requested object plus, recursively, every object it transitively
  references (via queued `read_pointer()` calls), continuing until the
  nesting level returns to where it started (bam ≥6.21) or the legacy
  `_num_extra_objects` counter reaches zero (older files). The returned
  object's pointers are **not** guaranteed complete yet — `resolve()` must
  be called afterward.
- **`resolve()` runs `complete_pointers()` in repeated passes**, since one
  object's `complete_pointers()` may itself be blocked on another object
  that hasn't completed yet; it loops calling
  `TypedWritable::complete_pointers()` on every still-pending object until
  either everything resolves or a pass makes no progress, then calls the
  private `finalize()` step (which drains everything registered via
  `register_finalize()`, respecting `finalize_now()` ordering requests made
  from within another object's `finalize()`).
- **Forward references are the whole reason `read_pointer()` doesn't return
  a pointer.** It queues a request (object ID) against the *currently
  reading* object (`_now_creating`) and returns only whether the pointer was
  non-null; actual pointers are delivered later, one per call in order, via
  the `complete_pointers(TypedWritable **p_list, ...)` array. `skip_pointer()`
  reads and discards an ID without registering a completion request.
- **`register_change_this()` lets an object swap itself out after
  construction**, e.g. to collapse into a shared/canonical instance. The
  registered `ChangeThisFunc`/`ChangeThisRefFunc` is invoked immediately
  after `complete_pointers()` for that object; every other object's
  in-flight pointer references to the old address are transparently
  redirected to the new one (`_created_objs_by_pointer` reverse index).
  Two overloads exist — a plain-`TypedWritable` variant and a
  `TypedWritableReferenceCount`-returning variant — because only the latter
  can safely destroy the old pointer inside the callback.
- **`change_pointer()`** is the caller-facing (not object-self-service)
  version of the same redirection — useful when external code decides after
  the fact that a just-read object should be replaced.
- **TypeHandles are interned per-stream, not per-object.** `read_handle()`
  reads a small integer index; the first occurrence of a given type is
  followed by its name and parent-type list (recursively `read_handle()`d),
  and later occurrences of the same index reuse the cached `TypeHandle`. An
  unrecognized type name gets dynamically registered on the fly (with a
  warning) so the file can still be parsed structurally even if the running
  program doesn't know that class.
- **PTA (PointerToArray) dedup works like pointer resolution but is
  synchronous.** `get_pta()` returns the already-registered array pointer
  for a previously-seen ID, or `nullptr` (and remembers the pending ID in
  `_pta_id`) the first time — the caller must then read the array data and
  call `register_pta()` with the freshly filled-in pointer. The `READ_PTA()`
  macro in `bamReader.h` packages this pattern.
- **`get_int_tag()`/`get_aux_tag()` are per-current-object scratch storage**
  set via `set_int_tag()`/`set_aux_tag()` during `fillin()` and read back
  during that same object's later `complete_pointers()` — useful for stashing
  a value read early that's only needed once pointers resolve.
- **`set_aux_data()`/`get_aux_data()` (distinct from int/aux *tags* above)**
  piggyback arbitrary `BamReader::AuxData`-derived state onto *any* object
  (or globally, if `obj == nullptr`) for the reader's own bookkeeping across
  the whole read pass, not tied to one object's `fillin()` call.
- **Object and PTA IDs upgrade from 16-bit to 32-bit mid-stream**: once an
  ID value of `0xffff` is seen, `_long_object_id`/`_long_pta_id` flips and
  all subsequent IDs of that kind are read as `uint32` — this must exactly
  mirror `BamWriter`'s same escalation logic.
- **`register_factory()`/`get_factory()` expose one process-wide
  `WritableFactory` (`Factory<TypedWritable>`)**, lazily created — every
  bam-serializable class registers its `make_from_bam`-style function here,
  typically from `init_type()`.

## API

### Core read loop
| Signature | Notes |
|---|---|
| `explicit BamReader(DatagramGenerator *source = nullptr)` | |
| `void set_source(DatagramGenerator*)` / `get_source()` | Also calls `init()` implicitly if needed |
| `bool init()` | Reads/validates the file header (major/minor version, endianness) |
| `TypedWritable *read_object()` | Single-pointer convenience form |
| `bool read_object(TypedWritable *&ptr, ReferenceCount *&ref_ptr)` | Preferred form — keeps ref-counting sound regardless of type |
| `bool resolve()` | Completes all pending pointers (may need repeated calls); runs `finalize()` when fully done |
| `bool is_eof() const` | Only meaningful after a `read_object()` call |
| `bool change_pointer(const TypedWritable *orig, const TypedWritable *new_ptr)` | Caller-driven pointer substitution |

### Object self-service (called from `fillin()`/`complete_pointers()`)
| Signature | Notes |
|---|---|
| `bool read_pointer(DatagramIterator&)` | Queues a pointer request; returns whether it was non-null |
| `void read_pointers(DatagramIterator&, int count)` | `count`× `read_pointer()` |
| `void skip_pointer(DatagramIterator&)` | Reads and discards, no completion request |
| `void read_file_data(SubfileInfo &info)` | Claims a block queued by a matching `write_file_data()` |
| `void read_cdata(DatagramIterator&, PipelineCyclerBase&)` / `(..., void *extra_data)` | Reads a pipelined `CycleData` via its `fillin()` |
| `void set_int_tag(const std::string&, int)` / `int get_int_tag(const std::string&) const` | Per-object scratch value, `fillin()` → `complete_pointers()` |
| `void set_aux_tag(...)` / `BamReaderAuxData *get_aux_tag(...) const` | Same, for a refcounted `BamReaderAuxData` payload |
| `void register_finalize(TypedWritable*)` | Requests a `finalize()` callback once all pointers resolve |
| `void finalize_now(TypedWritable*)` | Forces early finalization (for ordering, from within another's `finalize()`) |
| `void register_change_this(ChangeThisFunc, TypedWritable*)` / `(ChangeThisRefFunc, TypedWritableReferenceCount*)` | Post-construction self-replacement |
| `void *get_pta(DatagramIterator&)` / `void register_pta(void *ptr)` | PTA dedup (see `READ_PTA()` macro) |
| `TypeHandle read_handle(DatagramIterator&)` | Reads an interned `TypeHandle` |

### Reader/file state
| Signature | Notes |
|---|---|
| `const Filename &get_filename() const` | For interpreting relative paths inside the bam |
| `const LoaderOptions &get_loader_options() const` / `set_loader_options(...)` | |
| `int get_file_major_ver() const` / `get_file_minor_ver() const` | Must match/be supported by `get_current_major_ver()`/`get_current_minor_ver()` |
| `BamEndian get_file_endian() const` / `bool get_file_stdfloat_double() const` | |
| `void set_aux_data(TypedWritable *obj_or_null, const std::string &name, AuxData*)` / `AuxData *get_aux_data(...) const` | Reader-wide scratch storage, keyed by object+name |

### Aux data / factory
| Signature | Notes |
|---|---|
| `class BamReader::AuxData : ReferenceCount` | Subclass to attach reader-lifetime data via `set_aux_data()` |
| `class BamReaderAuxData : TypedReferenceCount` | Subclass for `set_aux_tag()`-style per-fillin data |
| `static void register_factory(TypeHandle, WritableFactory::CreateFunc*, void *user_data = nullptr)` | |
| `static WritableFactory *get_factory()` | `WritableFactory = Factory<TypedWritable>`, lazily created |

## Usage

```cpp
BamReader reader(&generator);
if (!reader.init()) { /* bad header */ }

TypedWritable *ptr; ReferenceCount *ref_ptr;
if (reader.read_object(ptr, ref_ptr) && reader.resolve()) {
  PT(MyClass) obj = DCAST(MyClass, ptr);
  // obj and everything it (transitively) references is now fully resolved
}
```

## See also

[BamWriter.md](BamWriter.md) (the write side — versioning, `BamEndian`
escalation, and ID-upgrade logic must match) · [TypedWritable.md](TypedWritable.md)
(the `fillin()`/`complete_pointers()`/`finalize()` contract) ·
[Factory.md](Factory.md) (`WritableFactory` mechanics) ·
[BamCache.md](BamCache.md) (disk-cache layer that sits above this) ·
[README.md](README.md)
