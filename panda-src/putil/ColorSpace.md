# ColorSpace

**Source:** `panda/src/putil/colorSpace.h` / `.cxx`
**Inherits:** (none — plain enum)
**Inherited by:** (n/a)

A global enum identifying the color space an image or color value is
encoded in, plus free-function string conversion helpers. Used throughout
the texture/image pipeline (`Texture`, `PNMImage`, etc. — outside this
module) to track whether pixel data is linear, sRGB-gamma-encoded, or
scRGB, so it can be converted correctly when it reaches the (linear)
graphics API.

```cpp
enum ColorSpace {
  CS_unspecified = 0,  // no color space given
  CS_linear,           // Panda's internal working space; all colors are
                        // linear unless stated otherwise
  CS_sRGB,              // standard gamma-2.2-ish sRGB, most image formats
  CS_scRGB,             // 16-bit linear, encodes values in [-0.5, 7.4999]
};
```

## Behavior notes

- **`CS_linear` is Panda's default assumption**, not merely one option among
  equals — per the header comment, "all colors in Panda3D are linear unless
  otherwise specified," and modern graphics APIs don't do color management
  themselves, so this enum exists purely for Panda's own bookkeeping (e.g.
  choosing whether to apply an sRGB-to-linear conversion on texture upload).
- **`parse_color_space_string()` accepts multiple aliases**: `"linear"`,
  `"linear-rgb"`, and `"lrgb"` all map to `CS_linear`; `"srgb"` (any case) to
  `CS_sRGB`; `"scrgb"` to `CS_scRGB`. `"non-color"` is also currently
  accepted and mapped to `CS_linear` — the source comment notes this is a
  placeholder "in case we want to add this as an enum value in the future."
- An unrecognized string logs an error (`util` notify category) and falls
  back to `CS_linear`, not `CS_unspecified`.
- `format_color_space(cs)` is just `operator<<` piped through an
  `ostringstream` — same output as streaming the enum directly.
- `operator<<` asserts (`nassertr`) on an out-of-range enum value rather
  than silently printing a placeholder like the other enum-streaming
  operators in this module do.

## API

| Signature | Notes |
|---|---|
| `ColorSpace parse_color_space_string(const std::string&)` | Also used by `operator>>` |
| `std::string format_color_space(ColorSpace)` | Same as `operator<<` |
| `std::ostream& operator<<(std::ostream&, ColorSpace)` | |
| `std::istream& operator>>(std::istream&, ColorSpace&)` | Delegates to `parse_color_space_string` |

## See also

[README.md](README.md)
