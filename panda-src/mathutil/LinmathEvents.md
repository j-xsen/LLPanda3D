# linmath_events (EventStoreVec2 / EventStoreVec3 / EventStoreMat4)

**Source:** `panda/src/mathutil/linmath_events.h` (+ `.cxx`, which is empty
beyond the `#include`)
**Inherits/Inherited by:** n/a — three typedefs, no new class

Despite the name suggesting event-system integration logic, this file
defines **no behavior at all** — it's three backward-compatibility typedef
aliases to types already fully defined in
[putil](../putil/ParamValue.md)'s `paramValue.h`:

```cpp
typedef ParamVecBase2 EventStoreVec2;
typedef ParamVecBase3 EventStoreVec3;
typedef ParamMatrix4  EventStoreMat4;
```

The header comment explains why: these `EventStoreX` names predate
[putil](../putil/ParamValue.md)'s generic `ParamValue<Type>` template family
and were the original classes used to stuff a linmath vector/matrix into an
`EventParameter` (the payload of an [event](../event/README.md) system
`Event`/`ButtonEvent`-style notification). When the engine's event-parameter
system was generalized to `ParamValue<Type>`, the old `EventStoreVec2`/
`EventStoreVec3`/`EventStoreMat4` class names were kept as plain typedefs to
the new `ParamVecBase2`/`ParamVecBase3`/`ParamMatrix4` types, purely so
existing call sites using the old names didn't need to change.

The `.cxx` file contains nothing but `#include "linmath_events.h"` — there
is no implementation to speak of.

## API

| Typedef | Aliases (from [putil/ParamValue.md](../putil/ParamValue.md)) |
|---|---|
| `EventStoreVec2` | `ParamVecBase2` |
| `EventStoreVec3` | `ParamVecBase3` |
| `EventStoreMat4` | `ParamMatrix4` |

## Usage

Prefer the modern names (`ParamVecBase2`/`ParamVecBase3`/`ParamMatrix4`,
documented in [../putil/ParamValue.md](../putil/ParamValue.md)) in new code;
`EventStoreVec2`/`3`/`EventStoreMat4` exist only so old code referencing
them still compiles.

## See also

[../putil/ParamValue.md](../putil/ParamValue.md) · [README.md](README.md)
