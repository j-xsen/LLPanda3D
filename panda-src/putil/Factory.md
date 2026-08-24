# Factory / FactoryBase / FactoryParam / FactoryParams

**Source:** `panda/src/putil/factory.h` / `.I` (template) + `factoryBase.h` / `.I` / `.cxx`
+ `factoryParam.h` / `.I` / `.cxx` + `factoryParams.h` / `.I` / `.cxx`
**Inherits:** `Factory<Type> : FactoryBase`; `FactoryParam : TypedReferenceCount`

A generic **type-registry-driven object factory**: subclasses of some base
`Type` register a plain creation function keyed by `TypeHandle`; callers
later ask for an instance "of this type, or the nearest thing I can make."
This is the mechanism [BamReader](BamReader.md) uses to construct the right
concrete class while reading a `.bam` file — every serializable class calls
`register_factory()` from its `init_type()`/static-init path.

`Factory<Type>` is a thin, header-only template wrapper with no data members
of its own; `FactoryBase` (non-templated, exported from the DLL) does the
real work in terms of `TypedObject*`, and `Factory<Type>` just casts.

## Behavior notes

- **Two different "give me something" queries, not one.**
  `make_instance(handle, params)` wants the *exact* type, falling back to a
  **derived** type only if the exact one isn't registered (walking a
  `_preferred` list first, then any registered type that derives from
  `handle`, in creator-map order). `make_instance_more_general(handle,
  params)` does the opposite: if the exact type isn't registered, it walks
  *up* the type's parent classes recursively until it finds one that is —
  this is what `BamReader` uses, since a bam file may reference a type the
  reading program doesn't know how to construct but does know a base class
  of. Don't confuse the two; they search in opposite directions.
- **`_preferred` only affects `make_instance()`'s specific-fallback path**,
  not the exact match and not `make_instance_more_general()` at all — it's a
  priority list of derived types to try first when the exact type can't be
  made.
- **A `Creator` bundles a raw C function pointer with a `void *user_data`**
  that gets stashed into `FactoryParams::_user_data` right before the create
  function is invoked (see `make_instance_exact()`) — this is how
  `register_factory(handle, func, user_data)` lets one function serve
  multiple registered types that need to know which one was actually
  requested.
- **`find_registered_type()`** performs the same "walk to a known ancestor"
  search as `make_instance_more_general()`, but only to answer "what's the
  nearest constructible type," without actually constructing anything —
  `BamWriter::flush_queue()` uses it to warn ahead of time if it's about to
  write an object of a type the reader side won't be able to reconstruct.
- **`FactoryParam` itself carries no data** — it exists only so concrete
  parameter types (e.g. [`BamReaderParam`](BamReader.md)) have a common,
  `TypedReferenceCount`-derived base that `FactoryParams` can store
  polymorphically and that a `CreateFunc` can `DCAST` back down from.
- **`FactoryParams::get_param_of_type()` prefers an exact type match, then
  falls back to the first param that merely derives from the requested
  type** — checked in two full passes over the params list, not
  interleaved. `get_param_into<ParamType>()` is a convenience wrapper around
  this that fills a typed pointer and returns whether it found anything.
- **`FactoryParams` is move-only-friendly but copyable** (`= default` copy,
  move, and move-assign) — it's just a `pvector<PT(TypedReferenceCount)>`
  plus a raw `void *_user_data`, so copies are cheap refcount bumps.

## API

### Factory\<Type> (template, wraps FactoryBase)
| Signature | Notes |
|---|---|
| `Type *make_instance(TypeHandle, const FactoryParams& = {})` | Exact type, else nearest **derived** (preferred-list first) |
| `Type *make_instance(const std::string &type_name, ...)` | Same, by registered type name |
| `Type *make_instance_more_general(TypeHandle, ...)` | Exact type, else nearest registered **ancestor** |
| `Type *make_instance_more_general(const std::string &type_name, ...)` | Same, by name |
| `void register_factory(TypeHandle, CreateFunc*, void *user_data = nullptr)` | `CreateFunc = Type *(*)(const FactoryParams&)` |

### FactoryBase
| Signature | Notes |
|---|---|
| `TypedObject *make_instance(TypeHandle, const FactoryParams&)` | Untyped version of the above |
| `TypedObject *make_instance_more_general(TypeHandle, const FactoryParams&)` | |
| `TypeHandle find_registered_type(TypeHandle)` | Nearest registered type without constructing |
| `void register_factory(TypeHandle, BaseCreateFunc*, void *user_data = nullptr)` | `BaseCreateFunc = TypedObject *(*)(const FactoryParams&)` |
| `size_t get_num_types() const` / `TypeHandle get_type(size_t n) const` | Debug enumeration of registered types (`get_type` is O(n)) |
| `void clear_preferred()` / `void add_preferred(TypeHandle)` / `get_num_preferred()` / `get_preferred(size_t)` | Priority list used by the specific-fallback path |
| `void write_types(std::ostream&, int indent_level = 0) const` | Debug dump |

### FactoryParam
Empty tag base — no members beyond `TypeHandle` plumbing. Subclass and add
data (see [`BamReaderParam`](BamReader.md)).

### FactoryParams
| Signature | Notes |
|---|---|
| `void add_param(FactoryParam*)` | Appends (ref-counted via `PT`) |
| `void clear()` | |
| `int get_num_params() const` / `FactoryParam *get_param(int n) const` | |
| `FactoryParam *get_param_of_type(TypeHandle) const` | Exact match first, then derived |
| `void *get_user_data() const` | Set internally by `FactoryBase` from the matched `Creator` |
| `template<class ParamType> bool get_param_into(ParamType *&, const FactoryParams&)` | Free function; fills pointer + returns found/not-found |

## Usage

```cpp
// Registration, typically from MyClass::init_type() or a config_*.cxx:
BamReader::get_factory()->register_factory(
    MyClass::get_class_type(), MyClass::make_from_bam);

// A class's factory create function:
static TypedWritable *make_from_bam(const FactoryParams &params) {
  MyClass *obj = new MyClass;
  DatagramIterator scan;
  BamReader *manager;
  parse_params(params, scan, manager);   // BamReader.h helper
  obj->fillin(scan, manager);
  return obj;
}
```

## See also

[BamReader.md](BamReader.md) (the primary consumer — `WritableFactory =
Factory<TypedWritable>`) · [TypedWritable.md](TypedWritable.md) ·
[WritableParam.md](WritableParam.md) · [README.md](README.md)
