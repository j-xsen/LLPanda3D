# FontPool

**Source:** `panda/src/text/fontPool.h` (+ `.I`, `.cxx`)

Static, filename-keyed font-loading cache — the `TextFont`-flavored sibling
of `TexturePool`/`ModelPool`. All methods are static and delegate to a
lazily-constructed global singleton (`get_ptr()`).

## Behavior notes

- **The lookup key encodes an optional FreeType face index appended after a
  colon** (e.g. `"myfont.ttf:1"` for the second face in a multi-face font
  file). `lookup_filename()` scans backward from the end of the string over
  digits, and if it finds a preceding `:`, splits into filename + face index;
  otherwise face index defaults to `0`. The filename portion is then resolved
  against `get_model_path()` via `VirtualFileSystem`, and the *resolved* path
  is what gets rebuilt into `index_str`, the actual cache key — so two
  different relative-path spellings that resolve to the same file share one
  cache entry.
- **Static/dynamic font selection is by extension, with a fallback.**
  `ns_load_font()` treats an empty extension, `"egg"`, or `"bam"` as a model
  file and tries `StaticTextFont` first. If that fails (file not found, or
  loaded but `!is_valid()`) *and* `HAVE_FREETYPE` is compiled in, it falls
  back to `DynamicTextFont` regardless of extension — so a `.ttf` file
  (unrecognized extension, skips the `StaticTextFont` attempt) still ends up
  going through the FreeType path on the fallback branch.
- **Double-checked locking around the actual load.** The cache is checked
  under `_lock`, released for the (possibly slow) disk load, then re-checked
  under `_lock` before inserting — if another thread loaded the same font
  while this one was reading from disk, the first-inserted result wins and
  the second load is silently discarded (not an error, just wasted work).
- **`garbage_collect()` only frees fonts with `get_ref_count() == 1`** (i.e.
  held only by the pool itself) — the same convention as `TexturePool`. This
  is unrelated to `DynamicTextFont::garbage_collect()`, which frees individual
  *glyphs* within a still-loaded font; calling this method does nothing to a
  font's internal glyph cache.
- **`release_font()`/`release_all_fonts()` don't destroy fonts synchronously**
  — they just drop the pool's own reference; the `TextFont` object itself
  survives as long as anything else (e.g. a `TextNode`) still holds a `PT()`
  to it.

## API

| Signature | Notes |
|---|---|
| `bool has_font(const std::string &filename)` | True if previously loaded (successfully or not — see caveat below) |
| `bool verify_font(const std::string &filename)` | `load_font(filename) != nullptr` |
| `BLOCKING TextFont *load_font(const std::string &filename)` | Loads (or returns cached) font; `nullptr` on failure |
| `void add_font(const std::string &filename, TextFont *font)` | Registers an already-constructed font under a filename key, replacing any existing entry |
| `void release_font(const std::string &filename)` | Drops the pool's reference |
| `void release_all_fonts()` | Clears the whole cache |
| `int garbage_collect()` | Frees fonts with ref count 1; returns count freed |
| `void list_contents(std::ostream&)` / `static void write(std::ostream&)` | Debug dump: filename + ref count per entry |

## Usage

```cpp
PT(TextFont) font = FontPool::load_font("cmss12.egg");
if (font == nullptr) {
  // not found / failed to load
}
text_node->set_font(font);
```

## See also

[TextFont.md](TextFont.md) · [StaticTextFont.md](StaticTextFont.md) ·
[DynamicTextFont.md](DynamicTextFont.md) · [README.md](README.md)
