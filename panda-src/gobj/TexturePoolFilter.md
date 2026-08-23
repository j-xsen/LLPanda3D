# TexturePoolFilter

**Source:** `panda/src/gobj/texturePoolFilter.h` (+ `.I`, `.cxx`)
**Inherits:** TypedObject

Abstract base class (default implementations are no-ops) for plugging a
callback into every first-time texture load performed by
[TexturePool](TexturePool.md). Subclass it, override `pre_load`/`post_load`
as needed, and register an instance via
`TexturePool::get_global_ptr()->register_filter()`. Only real known
implementer historically is `TxaFileFilter` in `pandatool`.

## Behavior notes

- `pre_load()` runs **before** the disk read; the first registered filter
  (in registration order) whose `pre_load()` returns non-null wins — that
  returned `Texture` is used as-is and the normal disk load is skipped
  entirely for this request. The default implementation returns `nullptr`
  (defer to normal loading).
- `post_load()` runs **after** the disk read, once per registered filter,
  each chained into the next (filter N's return value is passed as filter
  N+1's `tex` argument) — used to post-process the freshly-loaded `Texture`
  (e.g. rewrite properties) and return the (possibly different) `Texture`
  that `TexturePool` should actually hand back to the caller. Default
  implementation just returns `tex` unchanged.
- Both callbacks fire only on the **first** load of a given filename through
  the pool — a texture later reloaded from disk (e.g. via
  `Texture::reload()`) does *not* re-invoke the filter chain. If a filter
  needs to survive that, the source comment recommends the filter itself
  call `tex->set_keep_ram_image(true)` on textures it touches.

## API

| Signature | Notes |
|---|---|
| `virtual PT(Texture) pre_load(const Filename &orig_filename, const Filename &orig_alpha_filename, int primary_file_num_channels, int alpha_file_channel, bool read_mipmaps, const LoaderOptions &options)` | Override to intercept/substitute a load. Default: `nullptr`. |
| `virtual PT(Texture) post_load(Texture *tex)` | Override to post-process a freshly-loaded texture. Default: returns `tex`. |
| `virtual void output(ostream&) const` | Default prints the type name. |

## Usage

```cpp
class MyFilter : public TexturePoolFilter {
public:
  PT(Texture) post_load(Texture *tex) override {
    tex->set_wrap_u(SamplerState::WM_repeat);
    return tex;
  }
};
TexturePool::get_global_ptr()->register_filter(new MyFilter);
```

## See also

- [TexturePool](TexturePool.md), [Texture](Texture.md)
