# CachedTypedWritableReferenceCount / NodeCachedReferenceCount

**Source:** `panda/src/putil/cachedTypedWritableReferenceCount.h` / `.I` / `.cxx`
+ `nodeCachedReferenceCount.h` / `.I` / `.cxx`
**Inherits:** `CachedTypedWritableReferenceCount : `[`TypedWritableReferenceCount`](TypedWritable.md) ;
`NodeCachedReferenceCount : CachedTypedWritableReferenceCount`
**Inherited by:** [`CopyOnWriteObject`](CopyOnWriteObject.md); `NodeCachedReferenceCount`
is the base for `RenderState`/`TransformState` (in `pgraph`, not documented
here)

Adds **secondary reference counters that are subsets of, and automatically
maintained alongside, the primary `ReferenceCount`** — used to distinguish
*why* something is still alive (held by an ordinary cache, held by a
`PandaNode`, or held by neither and thus safe to evict) without needing a
separate weak-reference or observer mechanism.

- `CachedTypedWritableReferenceCount` adds one extra counter: the **cache
  reference count**. `get_ref_count() == get_cache_ref_count()` means the
  object is referenced *only* by whatever cache holds it, not by any
  ordinary `PointerTo<>` — a signal a cache can use to decide an entry is
  evictable.
- `NodeCachedReferenceCount` adds a second, independent extra counter on
  top of that: the **node reference count**, for objects (notably
  `RenderState`/`TransformState`) that are pointed to both by a cache *and*
  by `PandaNode`s directly, where PStats wants to distinguish "referenced by
  a node" from "referenced only by the cache" from "referenced by neither."

## Behavior notes

- **Every extra counter piggybacks on, rather than replaces, the base
  `ref()`/`unref()`.** `cache_ref()` calls `ref()` and increments
  `_cache_ref_count` together; `cache_unref()` decrements
  `_cache_ref_count` and calls the base `unref()` together. There is no
  independent lifetime tracking per counter — deleting still happens only
  when the *primary* count reaches zero (which happens automatically as a
  side effect, since every `cache_ref()`/`node_ref()` also bumped it).
- **The extra counts must be maintained explicitly — there is no automatic
  smart pointer for the cache count** (the class comment says so directly).
  `NodeCachedReferenceCount`, by contrast, *does* have a `NodePointerTo<>`
  smart-pointer counterpart (defined elsewhere) that calls `node_ref()`/
  `node_unref()` automatically — the cache-count/node-count asymmetry is
  deliberate, reflecting how each is actually used in practice.
- **`*_ref_only()`/`*_unref_only()` variants exist for subclasses that need
  to reimplement the outer `ref()`/`unref()` themselves** (see
  [`CopyOnWriteObject::unref()`](CopyOnWriteObject.md#behavior-notes), which
  overrides `unref()` and needs to touch the cache count without
  recursively calling back into `cache_unref()`). Don't call these directly
  from ordinary code — "Don't use this" is stated verbatim in the header
  for `cache_ref_only()`.
- **`NodeCachedReferenceCount` deliberately does NOT multiply-inherit from
  `NodeReferenceCount`** — the header explicitly says it duplicates that
  class's counting logic instead, specifically to avoid multiple-inheritance
  complications, since it must also inherit from
  `CachedTypedWritableReferenceCount`.
- **`get_referenced_bits()` (`NodeCachedReferenceCount` only)** returns a
  bitmask (`R_node = 0x001`, `R_cache = 0x002`) summarizing which kinds of
  references currently exist — an object with neither bit set is referenced
  by neither a node nor a cache (e.g. only by a transient local `PT()`), and
  is what PStats categorizes separately from the other two cases.
- **Constructors/destructors/assignment are `protected`**, same convention
  as plain `ReferenceCount` — you're not meant to instantiate these classes
  directly, only derive from them, and copies never inherit the source's
  reference counts (each starts its extra counters at 0).
- **Debug builds poison the counters to `-100` on destruction** and assert
  they're both `0` and never `-100` on every ref/unref, to catch use-after-
  free and double-delete bugs — same pattern as base `ReferenceCount`.
- **`cache_unref_delete<RefCountType>(ptr)`** (free template function,
  declared in `cachedTypedWritableReferenceCount.h`) is the
  `cache_ref()`-aware counterpart of `unref_delete()` — calls
  `ptr->cache_unref()` and deletes `ptr` only if that returned `false`
  (count reached zero). This is what [`CopyOnWritePointer`](CopyOnWriteObject.md)
  uses internally to release its held object.

## API

### CachedTypedWritableReferenceCount
| Signature | Notes |
|---|---|
| `int get_cache_ref_count() const` | |
| `void cache_ref() const` | `ref()` + increment cache count |
| `bool cache_unref() const` | decrement cache count + `unref()`; returns whether count is still nonzero |
| `bool test_ref_count_integrity() const` | Debug-only sanity check (no-op returning `true` in release builds) |
| `void cache_ref_only() const` *(protected-ish, "don't use this")* | Increments cache count without touching the primary count |

### NodeCachedReferenceCount (adds, on top of the above)
| Signature | Notes |
|---|---|
| `enum Referenced { R_node = 0x001, R_cache = 0x002 }` | |
| `int get_node_ref_count() const` | |
| `void node_ref() const` | `ref()` + increment node count |
| `bool node_unref() const` | decrement node count + `unref()` |
| `int get_referenced_bits() const` | OR of `R_node`/`R_cache` currently nonzero |

### Free function
| Signature | Notes |
|---|---|
| `template<class RefCountType> void cache_unref_delete(RefCountType *ptr)` | `cache_unref()`, then `delete` if count reached zero |

## See also

[CopyOnWriteObject.md](CopyOnWriteObject.md) (the main consumer of the cache
reference count) · [TypedWritable.md](TypedWritable.md) · [README.md](README.md)
