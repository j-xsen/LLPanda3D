# VertexDataSaveFile

**Source:** `panda/src/gobj/vertexDataSaveFile.h` (+ `.I`, `.cxx`)
**Inherits:** SimpleAllocator (see [SimpleAllocator.md](SimpleAllocator.md)) **Inherited by:** (none)

The single shared on-disk backing file that [`VertexDataPage`](VertexDataPage.md)s write themselves into when evicted to `RC_disk`. "All vertex data arrays are written into one large flat file" (per the header comment) — this is a `SimpleAllocator` over *file byte offsets* rather than memory, sub-allocating regions of one temp file the same way `VertexDataPage` sub-allocates regions of a memory buffer. Location and naming are controlled by the `vertex-save-file-directory`/`vertex-save-file-prefix` config vars (see the module README's config table); there is exactly one global instance, lazily created via `VertexDataPage::get_save_file()`.

## Behavior notes

- Platform-conditional file handle: a Win32 `HANDLE` under `_WIN32`, a POSIX file descriptor (`int _fd`) otherwise — this class talks to the OS file API directly rather than going through Panda's `VirtualFile`/filesystem abstraction, presumably for the low-level `pread`/`pwrite`-at-offset access pattern paging needs.
- `is_valid()` reports whether file creation actually succeeded (e.g. couldn't create in the configured directory) — callers (`VertexDataPage::do_save_to_disk()`) are expected to check this and fall back to keeping data in RAM if the save file isn't usable.
- Companion class `VertexDataSaveBlock` (declared in the same header) is this file's equivalent of `VertexDataBlock` — a `SimpleAllocatorBlock` representing one allocated byte range *within the save file*, plus a `_compressed` flag recording whether the bytes at that offset are stored deflated (since a page might already be compressed in RAM before it's written to disk, avoiding a decompress-then-recompress round trip).
- `write_data()`/`read_data()` are the only real entry points: `write_data()` allocates a block (via the inherited `SimpleAllocator`) and returns a ref-counted `VertexDataSaveBlock` handle to it; `read_data()` takes an existing block and reads its bytes back. Freeing the file space happens through the normal `SimpleAllocatorBlock` freeing mechanism (delete/`free()` the returned block), same pattern as `VertexDataBlock`.
- `_lock` (a `Mutex`, separate from the inherited `SimpleAllocator`'s own lock) protects the raw file I/O calls specifically, since multiple `PageThread`s may be reading/writing concurrently.

## API

| Signature | Notes |
|---|---|
| `VertexDataSaveFile(const Filename &directory, const std::string &prefix, size_t max_size)` | Creates (or opens) the backing temp file. |
| `bool is_valid() const` | Whether the file was successfully created/opened. |
| `size_t get_total_file_size() const` / `get_used_file_size() const` | Current allocated capacity vs. bytes actually in use. |
| `PT(VertexDataSaveBlock) write_data(const unsigned char *data, size_t size, bool compressed)` | Allocates space and writes `data`; records whether it's stored compressed. |
| `bool read_data(unsigned char *data, size_t size, VertexDataSaveBlock *block)` | Reads a previously-written block's bytes back into `data`. |

### `VertexDataSaveBlock`

| Signature | Notes |
|---|---|
| `void set_compressed(bool)` / `bool get_compressed() const` | Whether the stored bytes are zlib-compressed. |
| `unsigned char *get_pointer() const` | Not a real in-memory pointer for file-backed data in the usual sense — check the `.cxx`/`.I` if working at this level; primarily used internally by `VertexDataSaveFile`. |

## Usage

Not used directly by application code — reached only through [`VertexDataPage`](VertexDataPage.md)'s automatic disk-paging when memory pressure triggers `RC_disk` eviction.

## See also

- [VertexDataPage](VertexDataPage.md) — the class that actually decides when to page out to this file
- [SimpleAllocator](SimpleAllocator.md) — base class providing the byte-range allocation
