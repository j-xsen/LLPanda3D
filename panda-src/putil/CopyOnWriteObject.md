# CopyOnWriteObject / CopyOnWritePointer

**Source:** `panda/src/putil/copyOnWriteObject.h` / `.I` / `.cxx`
+ `copyOnWritePointer.h` / `.I` / `.cxx`
**Inherits:** `CopyOnWriteObject : `[`CachedTypedWritableReferenceCount`](CachedTypedWritableReferenceCount.md)
**Inherited by:** `CopyOnWriteObj<Base>`, `CopyOnWriteObj1<Base, Param1>` (template mixins, same header)

A copy-on-write smart-pointer pair: `CopyOnWriteObject` is the base class a
shareable payload derives from (directly, or via the `CopyOnWriteObj<Base>`/
`CopyOnWriteObj1<Base, Param1>` mixins that bolt COW semantics onto an
otherwise-plain type), and `CopyOnWritePointer` (or the typed
`CopyOnWritePointerTo<T>` / `COWPT(T)` alias) is the handle that negotiates
read vs. write access — multiple `CopyOnWritePointer`s may share one
underlying object until one of them asks to write, at which point it
transparently forks its own private copy.

## Behavior notes

- **Whether any of this actually locks depends on `HAVE_THREADS`.**
  `COW_THREADED` is defined exactly when `HAVE_THREADS` is; in a
  single-threaded build, `get_read_pointer()`/`get_write_pointer()` collapse
  to trivial inline accessors with **no locking at all** — only the
  copy-on-divergence behavior remains. Don't assume the mutex/condvar
  machinery exists; check `COW_THREADED` if you're touching this code.
- **The lock isn't a lock in the ordinary mutual-exclusion sense — it's a
  reader/writer negotiation over which *pointer* owns the object, gated on
  the *cache* reference count, not the plain reference count.** `cache_ref()`/
  `cache_unref()` (inherited machinery from
  [CachedTypedWritableReferenceCount](CachedTypedWritableReferenceCount.md))
  track how many `CopyOnWritePointer`s currently reference the object,
  separately from ordinary `ref()`/`unref()`. `CopyOnWriteObject::unref()`
  is overridden so that whenever the *plain* ref count drops down to equal
  the cache ref count (i.e. no ordinary `PT()` is holding it anymore, only
  COW pointers), the object is implicitly unlocked and any waiter is woken.
- **`get_write_pointer()` only copies when it must, and the "must" check is
  a genuine three-way branch** (threaded case, `copyOnWritePointer.cxx`):
  1. Another thread holds it **read-locked** → make a copy immediately (a
     reader might be mid-access).
  2. Nobody has it read-locked, but `cache_ref_count() > 1` (another
     `CopyOnWritePointer`, possibly on this same thread, also points at
     it) → also copy, since that other pointer's readers must not see this
     write.
  3. Otherwise (this is the only `CopyOnWritePointer` referencing it, and
     nobody else has it locked) → write in place, no copy.
  A copy is made via the pure-virtual `make_cow_copy()`, which every
  concrete leaf class must implement (`CopyOnWriteObj`/`CopyOnWriteObj1`
  supply it automatically via their template).
- **Same-thread re-entrant write-locking is allowed**: if the calling
  thread already holds the write lock (`_locking_thread == current_thread`),
  `get_write_pointer()` does not block on itself.
- **`get_read_pointer()` from the thread already holding the write lock
  returns the object without demoting the lock** — a writer can read its
  own in-progress write without triggering the reader-vs-writer wait.
- **`get_unsafe_pointer()` bypasses locking and copy-on-write entirely** —
  documented as usable "only in very narrow circumstances" where the caller
  independently knows no concurrent access is possible.
- **`CopyOnWritePointer` move-construction from a plain `PointerTo<T>`
  (not another `CopyOnWritePointer`) still has to `cache_ref_only()`** —
  since the source was an ordinary reference (not already counted in the
  cache-ref total), the move must add a cache reference rather than simply
  transferring one, even though it steals the underlying pointer and zeroes
  the source (`from.cheat() = nullptr`) to avoid a double-decrement.
- **`CopyOnWriteObj<Base>`/`CopyOnWriteObj1<Base, Param1>` register a
  synthetic `TypeHandle` at runtime** (`register_dynamic_type()`, name built
  from `typeid(Base).name()` when RTTI is available) rather than requiring a
  hand-written `init_type()` per instantiation — this is the same pattern
  `RefCountObj` uses for plain (non-COW) wrapped types, just layered on
  `CopyOnWriteObject` instead of `ReferenceCount`.
- **`PointerToBase<CopyOnWriteObject>::update_type()` is specialized to a
  no-op**, same rationale as the equivalent specialization in
  [TypedWritable.md](TypedWritable.md#behavior-notes) —
  `CopyOnWriteObject`'s type is already statically known.

## API

### CopyOnWriteObject
| Signature | Notes |
|---|---|
| `virtual PT(CopyOnWriteObject) make_cow_copy() = 0` | Protected; must be implemented by every concrete subclass |
| `bool unref() const` *(COW_THREADED only)* | Overridden to auto-unlock when ref count drops to the cache-ref count |
| `void cache_ref() const` / `bool cache_unref() const` *(COW_THREADED only)* | Mutex-guarded wrappers over the inherited cache-ref machinery |

### CopyOnWritePointer / CopyOnWritePointerTo\<T\>
| Signature | Notes |
|---|---|
| `CopyOnWritePointer(CopyOnWriteObject *object = nullptr)` | Also move-constructible from another `CopyOnWritePointer` or from `PointerTo<CopyOnWriteObject>&&` |
| `CPT(CopyOnWriteObject) get_read_pointer(Thread *current_thread) const` | Blocks if another thread holds the write lock (threaded build) |
| `PT(CopyOnWriteObject) get_write_pointer()` | May transparently copy — see behavior notes |
| `CopyOnWriteObject *get_unsafe_pointer()` | No locking, no copy — caller's responsibility |
| `bool is_null() const` / `void clear()` | |
| `bool test_ref_count_integrity() const` / `test_ref_count_nonzero() const` | Debug sanity checks |
| `operator ==` / `!=` / `<` | Identity comparison on the underlying pointer |

`CopyOnWritePointerTo<T>` (alias `COWPT(T)`) adds typed overloads of
`get_read_pointer()`/`get_write_pointer()`/`get_unsafe_pointer()` returning
`CPT(T)`/`PT(T)`/`T*` instead of the base-class pointer types.

## Usage

```cpp
class MyPayload : public CopyOnWriteObject {
protected:
  PT(CopyOnWriteObject) make_cow_copy() override {
    return new MyPayload(*this);
  }
  // ...
};

COWPT(MyPayload) shared = new MyPayload;
COWPT(MyPayload) alias = shared;   // shares the same object

{
  CPT(MyPayload) reader = shared.get_read_pointer(Thread::get_current_thread());
  // read-only access; `alias` still points at the same object
}

PT(MyPayload) writer = alias.get_write_pointer();
// since `shared` also references it, alias's object was just copy-on-written;
// `shared`'s data is untouched
```

## See also

[CachedTypedWritableReferenceCount.md](CachedTypedWritableReferenceCount.md)
(the cache-ref-count base this builds on) · [TypedWritable.md](TypedWritable.md) ·
[README.md](README.md)
