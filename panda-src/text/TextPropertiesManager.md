# TextPropertiesManager

**Source:** `panda/src/text/textPropertiesManager.h` (+ `.I`, `.cxx`)

Global registry mapping names to [TextProperties](TextProperties.md) and
[TextGraphic](TextGraphic.md) structures, so a text string can reference them
by name via the embedded `\x01`/`\x05` control characters — see
[README.md](README.md#core-concepts). One process-wide singleton
(`get_global_ptr()`); the constructor/destructor are protected specifically to
discourage creating a second instance.

## Behavior notes

- **Looking up an unregistered name silently creates a default entry and
  warns, rather than failing.** Both `get_properties(name)` and
  `get_graphic(name)`, on a miss, log a `text_cat.warning()`, insert a
  default-constructed structure under that name, and return it — so a typo'd
  name in embedded text doesn't throw or produce an "unknown" rendering, it
  just silently renders with default properties (or an empty/invisible
  graphic) from then on. `has_properties()`/`has_graphic()` distinguish "was
  explicitly registered" from "fell back to default" without triggering
  auto-creation.
- **The graphic-from-model convenience overload derives its frame from
  `calc_tight_bounds()`, not the model's declared bounding volume.**
  `set_graphic(name, model)` (without an explicit `TextGraphic`) computes the
  frame as the model's actual tight bounding box projected onto the X/Z
  (right/up) plane — more accurate but more expensive than the cheap
  bounding-volume overload. A hand-tuned frame (e.g. deliberately over- or
  under-sized) requires constructing a `TextGraphic` explicitly instead.

## API

### Properties
| Signature | Notes |
|---|---|
| `void set_properties(const std::string &name, const TextProperties&)` | Replaces any existing entry silently |
| `TextProperties get_properties(const std::string &name)` | Auto-creates + warns on miss; see notes |
| `bool has_properties(const std::string &name) const` | Doesn't trigger auto-creation |
| `void clear_properties(const std::string &name)` | No-op if not present |

### Graphics
| Signature | Notes |
|---|---|
| `void set_graphic(const std::string &name, const TextGraphic&)` | Explicit frame |
| `void set_graphic(const std::string &name, const NodePath &model)` | Frame derived from `calc_tight_bounds()`; see notes |
| `TextGraphic get_graphic(const std::string &name)` | Auto-creates + warns on miss |
| `bool has_graphic(const std::string &name) const` | — |
| `void clear_graphic(const std::string &name)` | — |

### Misc
| Signature | Notes |
|---|---|
| `static TextPropertiesManager *get_global_ptr()` | The one global instance |
| `void write(std::ostream&, int indent_level = 0) const` | Dumps all registered `TextProperties` |
| `const TextProperties *get_properties_ptr(const std::string&)` | Non-creating pointer lookup, used internally by `TextAssembler` |
| `const TextGraphic *get_graphic_ptr(const std::string&)` | Same, for graphics |

## Usage

```cpp
TextPropertiesManager *mgr = TextPropertiesManager::get_global_ptr();

TextProperties emphasis;
emphasis.set_text_color(1, 0, 0, 1);
mgr->set_properties("emphasis", emphasis);

// wtext contains: L"normal " + push_key + L"emphasis" + push_key +
//                 L"important" + pop_key + L" text"
```

## See also

[TextProperties.md](TextProperties.md) · [TextGraphic.md](TextGraphic.md) ·
[TextAssembler.md](TextAssembler.md) · [README.md](README.md)
