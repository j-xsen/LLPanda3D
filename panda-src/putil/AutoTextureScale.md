# AutoTextureScale

**Source:** `panda/src/putil/autoTextureScale.h` / `.cxx`
**Inherits:** (none — plain enum)
**Inherited by:** (n/a)

A small global enum controlling whether/how a texture's image is rescaled
(to a power-of-two or hardware-supported size) at load time. Used by
[LoaderOptions](LoaderOptions.md)'s `auto_texture_scale` property and by the
`textures-auto-power-2` / `auto-scale` family of prc config variables.

```cpp
enum AutoTextureScale {
  ATS_none,          // never rescale
  ATS_down,          // scale down to the nearest valid size
  ATS_up,            // scale up to the nearest valid size
  ATS_pad,           // pad with extra space instead of scaling
  ATS_unspecified,   // no explicit choice — fall back to the global default
};
```

## Behavior notes

- `operator>>` (string parsing) also accepts legacy boolean-style spellings
  for `none`/`down`: `"0"`, `"#f"`, or anything starting with `f`/`F` parses
  as `ATS_none`; `"1"`, `"#t"`, or anything starting with `t`/`T` parses as
  `ATS_down`. This is a compatibility shim from when the underlying config
  variable was a plain bool.
- An unrecognized string logs an error via the `util` notify category and
  falls back to `ATS_none`, rather than throwing or leaving the value
  uninitialized.
- `ATS_unspecified` is the sentinel `LoaderOptions` uses to mean "don't
  override the global default" — it round-trips through `operator<<` as the
  literal string `"unspecified"`, but `operator>>` does not parse that
  string back in (only `none`/`down`/`up`/`pad` and the boolean aliases are
  accepted as input).

## Usage

```cpp
LoaderOptions opts;
opts.set_auto_texture_scale(ATS_down);
```

## See also

[LoaderOptions.md](LoaderOptions.md) · [README.md](README.md)
