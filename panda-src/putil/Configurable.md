# Configurable / WritableConfigurable

**Source:** `panda/src/putil/configurable.h` / `.cxx` · `writableConfigurable.h` / `.cxx`
**Inherits:** `Configurable : TypedObject` · `WritableConfigurable : TypedWritable`

A tiny mixin base pair implementing a lazy "dirty config" pattern: derive
from one of these when an object has parameters that change rarely (far less
often than per-frame) but are expensive to recompute, so recomputation can
be deferred until the value is actually needed rather than redone on every
mutation. `WritableConfigurable` is a byte-for-byte duplicate of
`Configurable` except it derives from `TypedWritable` instead of
`TypedObject` and declares `write_datagram()` pure virtual — it exists
purely so a class needing to be both `Configurable` and bam-writable doesn't
hit an ambiguous-base-class error (`TypedWritable` and `TypedObject` are
both ancestors otherwise, and the compiler can't disambiguate). Pick
whichever base matches whether your subclass also needs `TypedWritable`.

## Behavior notes

- **Starts dirty.** Both constructors call `make_dirty()` immediately, so
  the first `check_config()` always triggers `config()` once.
- **`config()` must be overridden, and must call the base to clear the flag.**
  The base implementation only does `_dirty = false;` — a subclass overriding
  `config()` to do the actual recomputation work must still end by clearing
  dirty (either by calling the base `config()` or replicating `_dirty = false`),
  or `check_config()` will re-run `config()` on every call.
- **`check_config()` is `const` via a `const_cast`-style self-hack.** The
  source comment explains this deliberately: `check_config()` is meant to be
  callable from const accessor methods (e.g. a const "get the computed
  value" getter that lazily configures first), and the cast-away-const is
  justified as "not really modifying the class object" in spirit since it's
  just internal-consistency bookkeeping, not observable state.
- **`WritableConfigurable` registers `"WriteableConfigurable"` as an
  alternate type name** (`TypeRegistry::record_alternate_name()`) — a
  historical misspelling kept for backward Bam-file compatibility, so old
  `.bam` files that recorded the misspelled type name still resolve
  correctly.

## API

(Identical on both classes, aside from `WritableConfigurable`'s extra pure-virtual `write_datagram`.)

| Signature | Notes |
|---|---|
| `virtual void config()` | Override to recompute; must clear dirty |
| `void check_config() const` | Calls `config()` iff dirty, then returns |
| `bool is_dirty() const` | |
| `void make_dirty()` | Force the next `check_config()` to recompute |
| `virtual void write_datagram(BamWriter*, Datagram&) = 0` | `WritableConfigurable` only |

## Usage

```cpp
class MyThing : public Configurable {
public:
  const SomeComputedValue &get_value() const {
    check_config();          // lazily recompute if dirty
    return _cached_value;
  }
  void set_param(int p) {
    _param = p;
    make_dirty();            // defer recompute until next get_value()
  }
protected:
  virtual void config() override {
    _cached_value = expensive_compute(_param);
    Configurable::config();  // clears the dirty flag
  }
private:
  int _param = 0;
  SomeComputedValue _cached_value;
};
```

## See also

[README.md](README.md)
