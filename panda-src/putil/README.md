# PUtil — Bam Serialization, Bitmasks, Input Constants & Misc Utilities

**Source:** `panda/src/putil/` · Library: `libp3putil` · Notify category: `util`

`putil` is a grab-bag foundation module — it has no single theme, but three
clusters of it are load-bearing for nearly every other module in the engine:

1. **Bam serialization** — [TypedWritable](TypedWritable.md) is the base
   class of almost everything that can be saved to a `.bam` file (nodes,
   geoms, textures, animations...). [BamWriter](BamWriter.md) /
   [BamReader](BamReader.md) drive the write_datagram/fillin/
   complete_pointers/finalize lifecycle; [Factory](Factory.md) is how the
   reader turns a `TypeHandle` back into an instance; [BamCache](BamCache.md)
   is the on-disk cache of pre-loaded models keyed off this format.
2. **Bitmask/bit-set family** — [BitMask](BitMask.md) (fixed-width),
   [BitArray](BitArray.md) (dynamic/unbounded), and [SparseArray](SparseArray.md)
   (range-based sparse set) are the collision/draw/portal mask types used
   throughout [collide](../collide/README.md) and elsewhere.
3. **Input constants** — [ButtonHandle](ButtonHandle.md) +
   [ButtonRegistry](ButtonRegistry.md) + [Buttons](Buttons.md) (the
   `KeyboardButton`/`MouseButton`/`GamepadButton` namespaces) are how every
   input event in the engine names a physical button; [event](../event/README.md)'s
   `ButtonEvent` carries a `ButtonHandle`.

Everything else here — [ClockObject](ClockObject.md), small value types
([UpdateSeq](UpdateSeq.md), [TimedCycle](TimedCycle.md)), and internal
building blocks ([SimpleHashMap](SimpleHashMap.md),
[LinkedListNode](LinkedListNode.md), comparator templates) — is a standalone
utility used piecemeal by other modules.

## Class map

**Bam serialization:**
```
TypedObject
└── TypedWritable                          (TypedWritable.md)
    └── TypedWritableReferenceCount (: ReferenceCount)  (TypedWritable.md)
        └── CachedTypedWritableReferenceCount           (CachedTypedWritableReferenceCount.md)
            ├── NodeCachedReferenceCount                (CachedTypedWritableReferenceCount.md)
            └── CopyOnWriteObject                       (CopyOnWriteObject.md)
        └── FactoryParam (: TypedReferenceCount)        (Factory.md)
            └── WritableParam                           (WritableParam.md)
        └── ParamValueBase                               (ParamValue.md)
            ├── ParamTypedRefCount                       (ParamValue.md)
            └── ParamValue<Type>                         (ParamValue.md)

BamReader  (: BamEnums)                     (BamReader.md)  — reads a bam stream, resolves forward pointers
BamWriter  (: BamEnums)                     (BamWriter.md)  — writes a bam stream
BamReaderParam                              (BamReader.md)  — one arg passed to a make_from_bam() factory function
BamCache / BamCacheIndex / BamCacheRecord   (BamCache.md)   — on-disk cache of loaded/converted models
Factory<Type> (: FactoryBase)               (Factory.md)    — TypeHandle → instance-constructor registry
CopyOnWritePointer                          (CopyOnWriteObject.md) — smart pointer pairing with CopyOnWriteObject
DatagramInputFile / DatagramOutputFile / DatagramBuffer  (DatagramFile.md) — bam-stream sources/sinks
```

**Bitmask / bit-set family (no common base — three independent representations):**
```
BitMask<WType, nbits>          (BitMask.md)    — fixed-width, template; typedefs BitMask16/32/64
DoubleBitMask<BMType>          (BitMask.md)    — two BitMasks glued into one twice-as-wide value
DrawMask / CollideMask / PortalMask            — BitMask32 typedef aliases (BitMask.md)
BitArray                       (BitArray.md)   — dynamic, conceptually infinite bit array
SparseArray                    (SparseArray.md) — bit set stored as a sorted list of ranges
```

**Input constants:**
```
ButtonHandle                   (ButtonHandle.md)  — lightweight index into the button registry
ButtonRegistry                 (ButtonRegistry.md) — global name → ButtonHandle table
ButtonMap (: TypedReferenceCount) (ButtonRegistry.md) — physical-layout label overrides
KeyboardButton / MouseButton / GamepadButton   (Buttons.md) — static-namespace ButtonHandle constants
MouseData                      (Buttons.md)    — one mouse-position/in-window sample
ModifierButtons                (ModifierButtons.md) — tracks which watched buttons are currently held
```

**Callbacks / params / config:**
```
TypedReferenceCount
└── CallbackObject               (CallbackObject.md)
    └── CPointerCallbackObject   (CallbackObject.md)
    └── PythonCallbackObject     (not documented — Python interop only)
TypedObject
└── CallbackData                 (CallbackData.md)   — base of call-site-specific payload subclasses
Configurable (: TypedObject) / WritableConfigurable (: TypedWritable)  (Configurable.md)
```

**Everything else (independent utility classes):**
```
ClockObject (: ReferenceCount)  (ClockObject.md)  — global frame clock; real time vs. frame time
AnimInterface                   (AnimInterface.md) — play/loop/pose control-flow, base of AnimControl
AutoTextureScale / ColorSpace   (AutoTextureScale.md / ColorSpace.md) — small config enums
LoaderOptions                   (LoaderOptions.md) — flags object passed to loader calls
load_prc_file() / free functions (LoadPrcFile.md)  — runtime .prc config loading
SimpleHashMap<Key,Value>        (SimpleHashMap.md)  — Panda's internal open-addressing hash map
LinkedListNode                  (LinkedListNode.md) — intrusive doubly-linked-list mixin
NameUniquifier                  (NameUniquifier.md) — collision-avoiding name generator
UniqueIdAllocator               (UniqueIdAllocator.md) — recyclable small-int ID pool
UpdateSeq                       (UpdateSeq.md)      — monotonic "has this changed" sequence number
TimedCycle                      (TimedCycle.md)     — cycles through a value range over time
GlobalPointerRegistry / PointerData  (GlobalPointerRegistry.md) — unrelated pair sharing only a filename
Comparators (6 small templates) (Comparators.md)    — indirect-pointer / pair STL comparator function objects
Vectors and PTAs (8 typedef headers) (VectorsAndPTAs.md) — vector<T>/PointerToArray<T> instantiations + Datagram I/O
```

Not documented here (out of scope for this C++ reference):
- **`*_ext.h/.cxx`** (`bamReader_ext`, `bamWriter_ext`, `typedWritable_ext`,
  `callbackObject_ext`, `bitMask_ext`, `bitArray_ext`, `doubleBitMask_ext`,
  `sparseArray_ext`) — Python `EXTENSION(...)` method implementations,
  Python-interop only.
- **`pythonCallbackObject.h/.cxx`**, **`paramPyObject.h/.cxx`** — Python-callable
  subclasses, noted in passing in [CallbackObject.md](CallbackObject.md) /
  [ParamValue.md](ParamValue.md) but not documented in full.
- **`test_bam*.cxx`, `test_filename.cxx`, `test_glob.cxx`,
  `test_linestream.cxx`, `test_uniqueIdAllocator.cxx`** — standalone
  test/demo programs, not library code.
- **`config_putil.h/.cxx`, `config_util.h`** — module config/init
  boilerplate (registers types, no runtime API of its own); its notify
  category (`util`) is noted above.
- **`p3putil_composite1/2.cxx`, `p3putil_ext_composite.cxx`** — build-system
  unity-build wrapper files, not real source.
- **`bam.h`** — umbrella include with version constants only; mentioned
  in [TypedWritable.md](TypedWritable.md), no API of its own.
- **`pbitops.h/.cxx`** — internal popcount/bit-scan helper library
  `BitMask`/`BitArray` build on; noted briefly in
  [BitMask.md](BitMask.md), not documented as its own API since it's
  low-level and not meant to be called directly by application code.
- **`iterator_types.h`** — trivial internal typedef header with no
  standalone API.

## Core concepts

**Bam is Panda's one binary serialization format, and it has two very
different use cases that share the same wire protocol.** The obvious one is
`.bam` model files on disk — [BamCache](BamCache.md) exists specifically to
avoid re-converting the same `.egg`/glTF/etc. source file into bam on every
load. The less obvious one is *live* serialization: streaming a scene graph
to a connected client, where [TypedWritable::mark_bam_modified()](TypedWritable.md)
and `BamWriter::consider_update()` re-transmit only what changed since the
last snapshot, using [UpdateSeq](UpdateSeq.md) to detect staleness cheaply.

**Every serializable class implements the same four-method lifecycle, and
all four default to no-ops.** `write_datagram()` writes; `fillin()` reads
back the primitive fields; `complete_pointers()` resolves pointers that
couldn't be filled in immediately because the pointed-to object hadn't been
read yet (bam supports forward/circular references this way); `finalize()`
is a still-later hook for setup that needs the *entire* file resolved, not
just this object's own pointers. See [TypedWritable.md](TypedWritable.md)
for the full protocol and [BamReader.md](BamReader.md) /
[BamWriter.md](BamWriter.md) for how the two sides drive it.

**An object is looked up by name, not by number, when the reader needs to
construct one from scratch.** [Factory](Factory.md) maps a `TypeHandle` to a
registered "make one of these from a `BamReaderParam`" function pointer —
this is how `BamReader` turns "the stream says the next object is type X"
into an actual `new X`, without `putil` needing to know about every
concrete class that will ever exist.

**Three bit-set representations exist because none of them is right for
everything.** [BitMask](BitMask.md) is a fixed-width value type (cheap,
stack-allocated, used for draw/collide/portal masks where the bit count is
known at compile time). [BitArray](BitArray.md) is unbounded and tracks
whether its infinite tail is conceptually all-on or all-off (useful for
"everything except these few bits"). [SparseArray](SparseArray.md) stores
contiguous *ranges* rather than individual bits, and is cheap when the set
consists of a few large runs rather than scattered individual bits. See each
doc's intro for when to reach for which.

**A button is a name, not a scancode.** [ButtonHandle](ButtonHandle.md) is
a lightweight index into the global [ButtonRegistry](ButtonRegistry.md)'s
name table; [Buttons.md](Buttons.md)'s `KeyboardButton`/`MouseButton`/
`GamepadButton` namespaces are just pre-registered `ButtonHandle` constants
with well-known names (`"a"`, `"mouse1"`, `"shift"`...). This is the same
`ButtonHandle` carried by [event](../event/README.md)'s `ButtonEvent`, and
watched by [ModifierButtons](ModifierButtons.md) to track which of a
selected set are currently held.

**`ClockObject` separates "what time is it" from "what time should this
frame think it is."** `get_real_time()` always advances at wall-clock speed;
`get_frame_time()` only advances when something calls `tick()`, and *how*
it advances depends on `Mode` (real-time passthrough, fixed-rate
non-real-time capture, frame-rate-limited, deterministically-forced, etc.).
See [ClockObject.md](ClockObject.md) for the full mode table — this is the
engine's one global notion of "now."

## File index

| Topic | Purpose |
|---|---|
| [TypedWritable.md](TypedWritable.md) | Base classes for anything bam-serializable; the write/read lifecycle |
| [BamReader.md](BamReader.md) | Reads a bam stream; resolves forward pointers; `BamEnums` |
| [BamWriter.md](BamWriter.md) | Writes a bam stream; live-update re-transmission |
| [BamCache.md](BamCache.md) | On-disk cache of pre-converted models |
| [Factory.md](Factory.md) | `TypeHandle` → instance-constructor registry |
| [WritableParam.md](WritableParam.md) | Minimal `FactoryParam` carrying just a `TypedWritable*` |
| [CopyOnWriteObject.md](CopyOnWriteObject.md) | Copy-on-write object + its managing smart pointer |
| [CachedTypedWritableReferenceCount.md](CachedTypedWritableReferenceCount.md) | Ref-counted base for cache-evictable objects |
| [DatagramFile.md](DatagramFile.md) | Bam-stream file/buffer sources and sinks |
| [BitMask.md](BitMask.md) | Fixed-width bitmask template (+ `DrawMask`/`CollideMask`/`PortalMask`) |
| [BitArray.md](BitArray.md) | Dynamic, conceptually-infinite bit array |
| [SparseArray.md](SparseArray.md) | Range-list-based sparse bit set |
| [ButtonHandle.md](ButtonHandle.md) | Lightweight index into the button registry |
| [ButtonRegistry.md](ButtonRegistry.md) | Global button name table + `ButtonMap` label overrides |
| [Buttons.md](Buttons.md) | Predefined `ButtonHandle` constants (keyboard/mouse/gamepad) + `MouseData` |
| [ModifierButtons.md](ModifierButtons.md) | Tracks up/down state of a watched button set |
| [CallbackObject.md](CallbackObject.md) | Base class for pluggable callback hooks |
| [CallbackData.md](CallbackData.md) | Base class for callback invocation payloads |
| [ParamValue.md](ParamValue.md) | Typed value wrapper usable as a generic/factory parameter |
| [Configurable.md](Configurable.md) | Tiny "dirty config" mixin base pair |
| [ClockObject.md](ClockObject.md) | Global frame clock; real time vs. frame time, timing modes |
| [AnimInterface.md](AnimInterface.md) | Shared play/loop/pose control-flow, base of `AnimControl` |
| [AutoTextureScale.md](AutoTextureScale.md) | Texture auto-scaling config enum |
| [ColorSpace.md](ColorSpace.md) | Color-space enum + conversion helpers |
| [LoaderOptions.md](LoaderOptions.md) | Flags object passed to loader calls |
| [LoadPrcFile.md](LoadPrcFile.md) | Runtime `.prc` config file loading |
| [SimpleHashMap.md](SimpleHashMap.md) | Internal open-addressing hash map template |
| [LinkedListNode.md](LinkedListNode.md) | Intrusive doubly-linked-list mixin base |
| [NameUniquifier.md](NameUniquifier.md) | Collision-avoiding name generator |
| [UniqueIdAllocator.md](UniqueIdAllocator.md) | Recyclable small-integer ID pool |
| [UpdateSeq.md](UpdateSeq.md) | Monotonic "has this changed" sequence number |
| [TimedCycle.md](TimedCycle.md) | Cycles through a value range over time |
| [GlobalPointerRegistry.md](GlobalPointerRegistry.md) | `TypeHandle → void*` registry + unrelated `PointerData` |
| [Comparators.md](Comparators.md) | Indirect-pointer / pair STL comparator function-object templates |
| [VectorsAndPTAs.md](VectorsAndPTAs.md) | `vector<T>`/`PointerToArray<T>` typedefs + their bam Datagram I/O |

## Status

putil — done (2026-08-23). See [../../README.md](../../README.md) for the
overall index across `panda/src/*` modules.
