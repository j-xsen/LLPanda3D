# ParamValue / ParamValueBase / ParamTypedRefCount

**Source:** `panda/src/putil/paramValue.h` / `.I` / `.cxx`
**Inherits:** `ParamValueBase : TypedWritableReferenceCount` ·
`ParamTypedRefCount : ParamValueBase` · `ParamValue<Type> : ParamValueBase`
**Related:** Not documented here — `paramPyObject.h/.cxx` (Python glue,
wraps a `PyObject*` the same way), `EventParameter` in the `event` module
(a similar type-erased value box used for `Event` payloads, but a distinct,
non-BamWritable type).

A generic wrapper turning a plain value type into a refcounted, bam-writable,
`TypedObject` — i.e. something that can be handed around as a
`TypedWritableReferenceCount*` and stored inside containers that require
that interface, such as an `EventParameter` or a `ShaderInput`. There are
three levels: `ParamValueBase` (untyped virtual base, just `output()` and
`get_value_type()`), `ParamTypedRefCount` (wraps an existing
`TypedReferenceCount*`, e.g. to pass an already-refcounted object through a
`ParamValueBase*`-typed slot without copying it), and the `ParamValue<Type>`
template (wraps a value of any bam-serializable `Type` by copy).

## Behavior notes

- **`ParamValue<Type>` is explicitly instantiated only for a fixed set of
  types** — `std::string`, `std::wstring`, the `LVecBase{2,3,4}{d,f,i}`
  family, and `LMatrix{3,4}{d,f}` — each with a corresponding `typedef`
  (`ParamString`, `ParamVecBase3f`, `ParamMatrix4d`, etc.), plus
  precision-agnostic aliases (`ParamVecBase3`, `ParamMatrix4`, ...) that
  resolve to the `f` or `d` variant depending on whether Panda was built
  with `STDFLOAT_DOUBLE`. Instantiating `ParamValue<SomeOtherType>` yourself
  will compile but won't have Bam (de)serialization wired up unless you also
  call `register_with_read_factory()` for it and it satisfies
  `generic_write_datagram`/`generic_read_datagram`.
- **`get_class_type()`/`init_type()` for the template use a dynamic type
  name.** Unlike ordinary Panda classes, `ParamValue<Type>::init_type()`
  takes a `type_name` parameter (default `"UndefinedParamValue"`) and calls
  `register_dynamic_type()` rather than the usual static `register_type()`
  — each instantiation needs its own runtime `TypeHandle` registered under a
  distinct name (e.g. via the `typedef`'d aliases), since C++ template
  instantiations don't automatically get distinct compile-time type names in
  Panda's RTTI system. `force_init_type()` is consequently a no-op stub here
  ("we can't do anything, since we don't have the class' type_name").
- **`set_value()` calls `mark_bam_modified()`** — writing through the setter
  (not just constructing) flags the object as needing to be re-persisted if
  it's tracked by a `BamCache`-aware system.
- **`ParamTypedRefCount` stores by `TypedReferenceCount*` (not
  `TypedWritableReferenceCount*`)** and is deliberately a separate, plain
  (non-templated) class rather than `ParamValue<PT(TypedReferenceCount)>` —
  the comment notes it exists specifically because `TypedReferenceCount` is
  a different base hierarchy than `TypedWritableReferenceCount`.

## API

### ParamValueBase
| Signature | Notes |
|---|---|
| `virtual TypeHandle get_value_type() const` | `TypeHandle::none()` at this level |
| `virtual void output(std::ostream&) const = 0` | Pure virtual |

### ParamTypedRefCount
| Signature | Notes |
|---|---|
| `ParamTypedRefCount(const TypedReferenceCount *value)` | |
| `TypedReferenceCount *get_value() const` | |
| `TypeHandle get_value_type() const` | Delegates to the wrapped object's `get_type()`, or `none()` if null |

### ParamValue\<Type\>
| Signature | Notes |
|---|---|
| `ParamValue(const Type &value)` | |
| `void set_value(const Type&)` | Calls `mark_bam_modified()` |
| `const Type &get_value() const` | |
| `TypeHandle get_value_type() const` | `get_type_handle(Type)` |
| `static void register_with_read_factory()` | Wires up `BamReader`'s factory for `make_from_bam` |
| Predefined aliases | `ParamString`, `ParamWstring`, `ParamVecBase{2,3,4}{d,f,i}`, `ParamMatrix{3,4}{d,f}`, and precision-agnostic `ParamVecBase{2,3,4}`/`ParamMatrix{3,4}` |

## Usage

```cpp
PT(ParamString) p = new ParamString("hello");
throw_event("my-event", EventParameter(p));  // implicit TypedWritableReferenceCount* ctor

// later, on the receiving end:
ParamValueBase *pv = DCAST(ParamValueBase, param.get_ptr());
std::cout << *pv << "\n";
```

## See also

[README.md](README.md)
