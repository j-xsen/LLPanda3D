# DatagramInputFile / DatagramOutputFile / DatagramBuffer

**Source:** `panda/src/putil/datagramInputFile.h` / `.I` / `.cxx`,
`datagramOutputFile.h` / `.I` / `.cxx`, `datagramBuffer.h` / `.I` / `.cxx`
**Inherits:** `DatagramInputFile : DatagramGenerator`;
`DatagramOutputFile : DatagramSink`;
`DatagramBuffer : DatagramSink, DatagramGenerator`

The three concrete implementations of the `DatagramGenerator`/`DatagramSink`
interfaces (declared elsewhere) that [BamReader](BamReader.md)/
[BamWriter](BamWriter.md) actually read from and write to: a length-prefixed
sequence of `Datagram`s, either on disk (`DatagramInputFile`/
`DatagramOutputFile`) or entirely in memory (`DatagramBuffer`, which is both
a generator and a sink — see [`TypedWritable::encode_to_bam_stream()`](TypedWritable.md)
for its main use case). All three share one wire format, so a buffer written
with `DatagramBuffer` can be read back with `DatagramInputFile` and vice
versa.

## Behavior notes

- **Every datagram is prefixed with its length as a little-endian
  integer**, always 32-bit unless the datagram is large enough to need
  64 bits, in which case the 32-bit field is written as the sentinel
  `0xFFFFFFFF` followed by a 64-bit length — all three classes implement
  this same escalation identically (mirrors the object/PTA ID escalation in
  [BamReader](BamReader.md)/[BamWriter](BamWriter.md), but is an
  independent mechanism). A `0` length is a special-cased empty datagram
  with no further bytes.
- **The optional header is a raw, unprefixed byte sequence written before
  any datagram**, via `write_header()`/`read_header()` — the caller decides
  its length and contents (e.g. the bam magic string); it is not itself a
  datagram and has no length prefix of its own. Both `write_header()` and
  `read_header()` assert they are not called after the first datagram has
  already been written/read.
- **`DatagramInputFile::get_datagram()` reads defensively against a
  corrupt length field**: it caps each individual read to 4 MiB at a time
  rather than allocating the full claimed size up front, "so we don't
  allocate potentially a few GBs of RAM only to find a truncated file." It
  also reads directly into the `Datagram`'s internal buffer (via
  `modify_array()`) to avoid an extra copy.
- **`DatagramInputFile::save_datagram()` supports the `BamReader`
  file-data-block feature without buffering large blocks into memory.** If
  the input is backed by a real file, it just records a `SubfileInfo`
  pointing at the byte range within that file (`_in->tellg()` +
  `seekg`-past); if the input is *not* file-based (e.g. reading from an
  arbitrary `istream`), it falls back to copying the block out to a
  freshly-created `TemporaryFile` so a `SubfileInfo` can still reference it
  by file. `DatagramOutputFile::copy_datagram()` is the write-side
  counterpart, streaming a source file's or `SubfileInfo`'s bytes into the
  output in 4 KiB chunks.
- **`DatagramInputFile`/`DatagramOutputFile` can wrap either a `Filename`
  (opened internally, `_owns_in`/`_owns_out` tracks that so `close()` knows
  whether to actually close the stream) or a caller-supplied `istream`/
  `ostream`** — useful for reading/writing through something other than a
  plain disk file (e.g. a `VirtualFile`, or an already-open stream) while
  keeping the same datagram framing.
- **`DatagramBuffer` implements the identical framing purely over a
  `vector_uchar`** — `put_datagram()`/`get_datagram()` grow/scan the buffer
  directly rather than going through an `iostream`; its `flush()` is
  documented as doing "absolutely nothing" (there's no external sink to
  flush to). `get_data()`/`set_data()`/`swap_data()` give direct access to
  the underlying bytes for handing off to something else (e.g.
  `encode_to_bam_stream()`'s output `vector_uchar`).
- **`is_eof()` is only meaningful immediately after a failed
  `read_header()`/`get_datagram()` call** — calling it beforehand (or after
  a successful read) doesn't tell you anything useful; this is stated
  explicitly in all three classes' docs and mirrors the same convention as
  [`BamReader::is_eof()`](BamReader.md).

## API

### DatagramInputFile
| Signature | Notes |
|---|---|
| `bool open(const FileReference*)` / `bool open(const Filename&)` / `bool open(std::istream&, const Filename& = {})` | |
| `void close()` | |
| `bool read_header(std::string&, size_t num_bytes)` | Must precede any `get_datagram()` call |
| `bool get_datagram(Datagram&)` override | |
| `bool save_datagram(SubfileInfo&)` override | Records a large out-of-band block without fully reading it |
| `bool is_eof()` / `bool is_error()` override | |
| `const Filename &get_filename()` / `time_t get_timestamp() const` | |
| `const FileReference *get_file()` / `VirtualFile *get_vfile()` / `std::streampos get_file_pos()` | |

### DatagramOutputFile
| Signature | Notes |
|---|---|
| `bool open(const FileReference*)` / `bool open(const Filename&)` / `bool open(std::ostream&, const Filename& = {})` | |
| `void close()` | |
| `bool write_header(const vector_uchar&)` / `bool write_header(const std::string&)` | Must precede any `put_datagram()` call |
| `bool put_datagram(const Datagram&)` override | |
| `bool copy_datagram(SubfileInfo &result, const Filename&)` / `(SubfileInfo &result, const SubfileInfo &source)` override | Streams a large block in from disk |
| `bool is_error()` override / `void flush()` override | |
| `const Filename &get_filename()` / `const FileReference *get_file()` / `std::streampos get_file_pos()` | |

### DatagramBuffer
| Signature | Notes |
|---|---|
| `DatagramBuffer()` / `explicit DatagramBuffer(vector_uchar data)` | |
| `void clear()` | |
| `bool write_header(const std::string&)` / `bool read_header(std::string&, size_t num_bytes)` | |
| `bool put_datagram(const Datagram&)` override / `void flush()` override (no-op) | |
| `bool get_datagram(Datagram&)` override / `bool is_eof()` override / `bool is_error()` override (always `false`) | |
| `const vector_uchar &get_data() const` / `void set_data(vector_uchar)` / `void swap_data(vector_uchar&)` | Direct buffer access |

## Usage

```cpp
DatagramOutputFile dout;
dout.open("data.bin");
dout.write_header("MYFMT");
dout.put_datagram(dg);
dout.close();

DatagramInputFile din;
din.open("data.bin");
std::string header;
din.read_header(header, 5);
Datagram dg2;
while (din.get_datagram(dg2)) { /* ... */ }
```

## See also

[BamReader.md](BamReader.md) / [BamWriter.md](BamWriter.md) (the primary
consumers — `DatagramGenerator`/`DatagramSink` are their source/target
interfaces) · [TypedWritable.md](TypedWritable.md#behavior-notes)
(`encode_to_bam_stream()` uses `DatagramBuffer` internally) ·
[README.md](README.md)
