# TransparencyAttrib

**Source:** `panda/src/pgraph/transparencyAttrib.h` (+ `.I`, `.cxx`)
**Inherits:** RenderAttrib

Enables and selects the mode of transparency rendering. Setting an alpha
component below 1 does **not** by itself make geometry transparent — a
`TransparencyAttrib` must also be set; conversely, setting one without any
sub-1 alpha is wasted GPU work.

## Behavior notes

- `M_none` and `M_alpha` are deliberately `0` and `1` — historically
  `NodePath::set_transparency()` took a plain `bool`, and those values had
  to line up.
- `M_alpha`: standard back-to-front sorted blending (see `CullBin`'s
  transparent bin in the [cull pipeline](README.md#cull-pipeline)).
  `M_premultiplied_alpha`: assumes the texture's RGB is already multiplied
  by its alpha (avoids a separate blend-mode switch for such textures).
  `M_multisample`/`M_multisample_mask`: alpha-to-coverage via the MSAA
  buffer instead of blending — `M_multisample` clamps alpha to 1.0 after
  using it for coverage, `M_multisample_mask` leaves it unmodified.
  `M_binary`: cheap cutout transparency, only pixels with alpha ≥ 0.5 are
  written (no sorting needed). `M_dual`: renders opaque parts of the
  object first, then the transparent parts sorted — see the
  `m-dual`/`m-dual-opaque`/`m-dual-transparent`/`m-dual-flash`
  [config variables](README.md#config-variables-from-config_pgraphhcxx).

## API

| Signature | Notes |
|---|---|
| `enum Mode` | `M_none`, `M_alpha`, `M_premultiplied_alpha`, `M_multisample`, `M_multisample_mask`, `M_binary`, `M_dual` |
| `static CPT(RenderAttrib) make(Mode mode)` | |
| `static CPT(RenderAttrib) make_default()` | `M_none` |
| `Mode get_mode() const` | |

## Usage

```cpp
node_path.set_transparency(TransparencyAttrib::M_alpha); // NodePath wrapper
```

## See also

[README — the state pipeline](README.md#the-state-pipeline),
[README — cull pipeline](README.md#cull-pipeline), [RenderAttrib](RenderAttrib.md)
