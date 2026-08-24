# EventParameter

**Source:** `panda/src/event/eventParameter.h` / `.I` / `.cxx`
**Inherits:** none (standalone value type wrapping a `PT(TypedWritableReferenceCount)`)

A type-erased, refcounted value carried by an [Event](Event.md). Cheap to
copy (it's a smart pointer). Transparently wraps the common cases —
int, double, `std::string`, `std::wstring`, or a
`TypedReferenceCount`/`TypedWritableReferenceCount` pointer — via the
`ParamValue<T>` template family from `paramValue.h`, so the wrapping is
rarely something calling code needs to be aware of.

## Behavior notes

- **Construct implicitly from any supported type** — the constructors are all
  non-`explicit`, so `EventParameter(42)`, `EventParameter("hi")`,
  `EventParameter(some_node_ptr)` all work directly, and `throw_event(name, 42)`
  works without explicit wrapping.
- **`EventParameter(TypedReferenceCount*)` vs. `EventParameter(TypedWritableReferenceCount*)`
  take genuinely different storage paths** — a `TypedReferenceCount*` gets
  boxed inside an `EventStoreTypedRefCount` wrapper (since it isn't itself a
  `TypedWritableReferenceCount`), while a `TypedWritableReferenceCount*` is
  stored directly. This case is distinguished via `is_typed_ref_count()`
  rather than by calling `get_ptr()` and inspecting its `TypeHandle` directly.
- **Constness is lost on pointer construction** — the pointer constructors
  accept `const T*` but store (and later return) non-`const T*`; this is a
  documented, deliberate simplification, not a bug — "be careful," per the
  header.
- **`get_ptr()`** is the escape hatch for parameter types that aren't one of
  the predefined int/double/string/wstring/typed-ref-count cases: it returns
  the raw `TypedWritableReferenceCount*`, requiring the caller to inspect
  `get_type()` / `is_of_type()` / cast (`DCAST`) directly.
- **A default-constructed or `nullptr`-constructed `EventParameter` is
  "empty"** (`is_empty()` true) — none of the `is_int()`/`is_double()`/etc.
  checks will pass on it.

## API

| Signature | Notes |
|---|---|
| `EventParameter()` / `EventParameter(nullptr)` | Empty parameter |
| `EventParameter(const TypedWritableReferenceCount*)` | Stores directly |
| `EventParameter(const TypedReferenceCount*)` | Boxed via `EventStoreTypedRefCount` |
| `EventParameter(int)` / `EventParameter(double)` / `EventParameter(const std::string&)` / `EventParameter(const std::wstring&)` | Boxed via `ParamValue<T>` / `ParamString` / `ParamWstring` |
| `bool is_empty() const` | |
| `bool is_int() const` / `int get_int_value() const` | `get_int_value()` asserts `is_int()` |
| `bool is_double() const` / `double get_double_value() const` | |
| `bool is_string() const` / `std::string get_string_value() const` | |
| `bool is_wstring() const` / `std::wstring get_wstring_value() const` | |
| `bool is_typed_ref_count() const` / `TypedReferenceCount *get_typed_ref_count_value() const` | For values from the `TypedReferenceCount*` constructor |
| `TypedWritableReferenceCount *get_ptr() const` | Raw escape hatch; inspect its `TypeHandle` for anything else |
| `void output(std::ostream&) const` | |

## Usage

```cpp
throw_event("score-changed", EventParameter(new_score), EventParameter(player_name));

void on_score_changed(const Event *event) {
  int score = event->get_parameter(0).get_int_value();
  std::string name = event->get_parameter(1).get_string_value();
}
```

## See also

[Event.md](Event.md) (holds a list of these) · [throw_event.md](throw_event.md) ·
[README.md](README.md)
