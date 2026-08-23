# BamFile

**Source:** `panda/src/pgraph/bamFile.h` (+ `.I`, `.cxx`) — skips Python glue `bamFile_ext.h/.cxx`
**Inherits:** `BamEnums`

The principal interface to reading and writing Panda's native binary object
format ("Bam" — `.bam`/`.boo` files). A thin, convenient wrapper around the
lower-level `BamReader`/`BamWriter`. Bam files store an arbitrary sequence
of `TypedWritable` objects; by convention a file storing a scene graph
(a single `PandaNode`) is named `.bam`, while one storing other object
sequences is named `.boo` ("binary other objects").

## Behavior notes

- One `BamFile` is single-direction per open: `open_read()` xor
  `open_write()` — `close()` (also called by the destructor) tears down
  whichever of `_reader`/`_writer` is active. Both file-based and
  arbitrary-`istream`/`ostream`-based open overloads exist.
- **`read_object()` returns pointers that are not yet safe to use** — call
  `resolve()` afterward to fix up all internal cross-references (a Bam file
  can contain forward/circular pointer references written before their
  targets, resolved in a second pass). `read_node()` calls `resolve()` for
  you.
- **`read_node()` is the convenience path for scene-graph bam files**: reads
  one object, verifies it's a `PandaNode` (errors and returns null
  otherwise), resolves it, and — if `report_errors` — reads once more to
  warn about any unexpected extra objects in the file. It also transparently
  skips a leading `BamCacheRecord` object if present (a bam file fetched
  from the on-disk `BamCache` is itself a cache-record-prefixed bam stream,
  indistinguishable from an ordinary model file to this method).
- `get_file_major_ver()`/`get_file_minor_ver()`/`get_file_endian()`/
  `get_file_stdfloat_double()` report properties of whichever stream is
  currently open (reader takes priority over writer if somehow both are
  set); with nothing open they report the *system's* current values instead
  (what would be written to a new file).
- `open_write()` on a filename silently deletes any existing file at that
  path first (via `VirtualFileSystem::delete_file()`).

## API

| Method | Notes |
|---|---|
| `bool open_read(Filename, report_errors=true)` / `open_read(istream&, name="stream", report_errors=true)` | |
| `TypedWritable *read_object()` | Pointers unsafe until `resolve()` |
| `bool is_eof() const` | Valid only after a `read_object()` call |
| `bool resolve()` | Fix up cross-references from prior `read_object()` calls; may need repeat calls |
| `PT(PandaNode) read_node(report_errors=true)` | Convenience: read+verify+resolve one scene-graph node |
| `bool open_write(Filename, report_errors=true)` / `open_write(ostream&, ...)` | Deletes existing file first |
| `bool write_object(const TypedWritable*)` | |
| `void close()` | Also called by destructor |
| `bool is_valid_read() const` / `is_valid_write() const` | |
| `int get_file_major_ver()` / `get_file_minor_ver()` | |
| `BamEndian get_file_endian() const` / `bool get_file_stdfloat_double() const` | |
| `int get_current_major_ver()` / `get_current_minor_ver()` | System's version, static-equivalent |
| `BamReader *get_reader()` / `BamWriter *get_writer()` | Null unless the corresponding open succeeded |

## Usage

```cpp
BamFile bam;
if (bam.open_read("model.bam")) {
  PT(PandaNode) root = bam.read_node();
  bam.close();
}

BamFile out;
if (out.open_write("saved.bam")) {
  out.write_object(some_node);
  out.close();
}
```

## See also

- [LoaderFileTypeBam](LoaderFileTypeBam.md) — the `Loader`-facing wrapper around this class
- [Loader](Loader.md)
