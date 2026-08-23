# TextureCollection

**Source:** `panda/src/gobj/textureCollection.h` (+ `.I`, `.cxx`; skip `textureCollection_ext.h/.cxx` — Python glue)
**Inherits:** (none — value type wrapping a `PTA(PT(Texture))`)

An ordered list of `Texture` pointers, as returned by e.g.
`TexturePool::find_all_textures()`. Same shape as `pgraph`'s
`NodePathCollection` (see [../pgraph/README.md](../pgraph/README.md)):
copied by value, backed by a ref-counted `PointerToArray`, cheap to pass
around.

## Behavior notes

- `add_texture()`/`append()` are aliases; `remove_texture()` returns
  `false` if the texture wasn't present. `remove_duplicate_textures()`
  dedupes by pointer identity, not by content/filename.
- `find_texture(name)` does a linear scan comparing `Texture::get_name()`;
  returns the first match or `nullptr`.
- `operator+`/`operator+=` concatenate two collections (union with possible
  duplicates — does not dedupe).

## API

| Signature | Notes |
|---|---|
| `TextureCollection()`, copy ctor, `operator=` | |
| `void add_texture(Texture*)` / `append(Texture*)` | |
| `bool remove_texture(Texture*)` | |
| `void add_textures_from(const TextureCollection&)` / `remove_textures_from(...)` / `extend(...)` | |
| `void remove_duplicate_textures()` | Pointer-identity dedupe. |
| `bool has_texture(Texture*) const` | |
| `void clear()` / `void reserve(size_t)` | |
| `Texture *find_texture(const string &name) const` | First match by name, linear scan. |
| `int get_num_textures() const`, `Texture *get_texture(int) const`, `operator[]`, `int size() const` | |
| `MAKE_SEQ(get_textures, ...)` | Python sequence protocol glue. |
| `operator+=` / `operator+` | Concatenation (no dedupe). |
| `void output(ostream&) const` / `void write(ostream&, int indent=0) const` | |

## See also

- [Texture](Texture.md), [TexturePool](TexturePool.md)
