# ConfigVariableColor

**Source:** `panda/src/linmath/configVariableColor.h/.I` · Library:
`libp3linmath` · Notify category: `linmath`
**Inherits:** `ConfigVariable`
**Inherited by:** (none)

A `ConfigVariable` specialization (see the prc config system) that reads its
value as an [`LColor`](LVecBase.md) (`= LVecBase4f`/`LVecBase4d`, aliased per
`STDFLOAT_DOUBLE`). It lives in `linmath` rather than `dtool` (where the
other `ConfigVariable*` specializations live) specifically because it
depends on `LColor`, which isn't available at `dtool`'s level of the build.

## Behavior notes

- **Word count in the prc declaration determines how the color is
  interpreted** (`get_value()`'s `switch` on `get_num_words()`): one word →
  grayscale with alpha forced to `1` (`(w,w,w,1)`); two words → grayscale +
  explicit alpha (`(w0,w0,w0,w1)`); three words → RGB with alpha forced to
  `1`; four words → full RGBA. An unexpected word count (0, or 5+) logs a
  `prc_cat->warning()` and leaves the cached value unchanged rather than
  erroring.
- **Values are cached and invalidated via the same `AtomicAdjust`-based
  local-modified-counter pattern other `ConfigVariable` subclasses use** —
  `get_value()` only re-parses `get_double_word(n)` calls when
  `is_cache_valid(_local_modified)` is false, then `mark_cache_valid()`s and
  returns the cached `LColor` by reference. `set_value(const LColor&)`
  writes back through `set_double_word()` per-component (clearing any string
  value first via `set_string_value("")`), which invalidates the cache for
  the next read.
- **`get_default_value()` re-derives its own default independently of
  `get_value()`'s caching** — it reads `ConfigVariable::get_default_value()`'s
  `ConfigDeclaration*` directly and applies the exact same
  1/2/3/4-word-count interpretation switch, rather than sharing code with
  `get_value()`. Returns `LColor(0, 0, 0, 1)` (opaque black) if no default
  declaration exists or its word count is invalid.
- **`operator[](int n)` indexes the *resolved* `LColor`'s components, not the
  raw config words** — per its doc comment, "not necessarily the same thing
  as the variable's nth word" (a one-word grayscale declaration has 4
  resolved components but only 1 stored word).
- **Construction from a string default** (`ConfigVariableColor(name,
  default_value_as_string, ...)`) goes through
  `_core->set_default_value(default_value)` as raw prc text, letting the
  normal prc parser (not this class) tokenize it into words — this is how a
  `.prc` file's `my-color 1 0 0` line ends up interpreted through the same
  word-count switch as a programmatically-set `LColor`.

## API

| Signature | Notes |
|---|---|
| `ConfigVariableColor(const std::string &name)` | No default; default value defaults to `(0,0,0,1)` until set |
| `ConfigVariableColor(const std::string &name, const LColor &default_value, const std::string &description = "", int flags = 0)` | |
| `ConfigVariableColor(const std::string &name, const std::string &default_value, const std::string &description = "", int flags = 0)` | Default parsed from raw prc text |
| `void operator=(const LColor &value)` / `operator const LColor &() const` | Implicit conversion to/from `LColor` |
| `PN_stdfloat operator[](int n) const` | Indexes the resolved color, not the raw words |
| `void set_value(const LColor &value)` / `const LColor &get_value() const` | Explicit accessors backing the operators above |
| `LColor get_default_value() const` | Independently re-parsed from the declared default; see behavior notes |

## Usage

```cpp
// prc file: "background-color 0.2 0.2 0.3 1.0"
ConfigVariableColor background_color("background-color", LColor(0, 0, 0, 1));

LColor bg = background_color;          // implicit conversion
float alpha = background_color[3];
background_color = LColor(1, 1, 1, 1); // override at runtime
```

## See also

[LVecBase.md](LVecBase.md) (`LColor` = `LVecBase4f`/`LVecBase4d`) ·
[LoadPrcFile.md](../putil/LoadPrcFile.md) (the prc config system this plugs
into) · [README.md](README.md)
