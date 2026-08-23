# CullBinAttrib

**Source:** `panda/src/pgraph/cullBinAttrib.h` (+ `.I`, `.cxx`)
**Inherits:** [RenderAttrib](RenderAttrib.md)

Assigns geometry to a named cull bin (opaque/transparent/fixed/etc.),
optionally with a draw order. The bin itself must already exist — created
separately via [CullBinManager](CullBinManager.md); this attrib only
references it by name.

## Behavior notes
- Empty `bin_name` means "use the default bin" — `make_default()` uses
  the parameterless constructor, which sets an empty bin name and draw
  order `0`.
- `draw_order` is only meaningful to bin types that use explicit ordering
  (notably `CullBinFixed`); other bin types ignore it.
- No custom `compose_impl`/`invert_compose_impl` — inherits `RenderAttrib`'s
  default "later attrib completely replaces" composition.

## API

| Method | Notes |
|---|---|
| `static CPT(RenderAttrib) make(const std::string &bin_name, int draw_order)` | |
| `static CPT(RenderAttrib) make_default()` | Empty name (default bin), draw order 0 |
| `const std::string &get_bin_name() const` | |
| `int get_draw_order() const` | |

## Usage
```cpp
node_path.set_attrib(CullBinAttrib::make("fixed", 10));
```

## See also
[RenderAttrib](RenderAttrib.md), [CullBinManager](CullBinManager.md),
[CullBin](CullBin.md), [pgraph README — the state pipeline](README.md#the-state-pipeline)
