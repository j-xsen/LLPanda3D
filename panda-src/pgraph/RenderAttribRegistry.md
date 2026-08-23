# RenderAttribRegistry

**Source:** `panda/src/pgraph/renderAttribRegistry.h` (+ `.I`, `.cxx`)
**Inherits:** (none)

Process-global singleton that assigns each concrete [RenderAttrib](RenderAttrib.md)
subclass a fixed slot index (0–31, capped by `_max_slots = 32` since slots
are tracked via a `BitMask32` = `SlotMask`) at static-init time, so
[RenderState](RenderState.md) can store attribs in a fixed-size array and
test presence with a bitmask instead of a hash lookup. Application code
essentially never touches this directly — it's plumbing behind
`RenderAttrib::register_slot()` (called once per subclass, typically from
that subclass's `init_type()`) and `RenderState::get_attrib(TypeHandle)`.

## Behavior notes

- **Slot exhaustion is a hard cap.** There are exactly 32 slots
  (`SlotMask` = `BitMask32`); the header comment notes that if Panda ever
  grows past 32 built-in `RenderAttrib` subclasses, `SlotMask` needs to
  widen to a 64-bit mask — this is a real ceiling, not an approximation.
- Slot 0 is reserved as "no slot assigned" — `get_slot()` returns `0` for
  an unregistered `TypeHandle`, so registered slots effectively start at 1.
- Each slot also carries a **sort number** (`set_slot_sort`/`get_slot_sort`)
  independent of the slot index — `get_sorted_slot(n)` walks slots in that
  sort order rather than allocation order, which is what code iterating
  "all attrib slots in a stable, meaningful order" (e.g. state output/
  comparison) should use instead of raw slot indices.
- Each slot also stores a **default attrib** (`get_slot_default`) — the
  attrib value that applies when a `RenderState` has no explicit attrib in
  that slot; this is what `RenderState::get_attrib_def()` falls back to.
- `get_global_ptr()` lazily constructs the singleton on first use;
  `quick_get_global_ptr()` skips the null check for hot-path callers that
  know it's already initialized (used by `RenderState`'s inline accessors).

## API

| Method | Notes |
|---|---|
| `int register_slot(type_handle, sort, default_attrib)` | Called once per RenderAttrib subclass; returns its assigned slot |
| `int get_slot(TypeHandle) const` | Slot for a type, or 0 if unregistered |
| `static constexpr int get_max_slots()` | 32 |
| `int get_num_slots() const` | Slots allocated so far (one past the highest in use) |
| `TypeHandle get_slot_type(int slot) const` | Reverse lookup |
| `int get_slot_sort(int slot) const` / `void set_slot_sort(int, int)` | Sort-order plumbing |
| `const RenderAttrib *get_slot_default(int slot) const` | Default value for the slot |
| `int get_num_sorted_slots() const` / `int get_sorted_slot(int n) const` | Iterate slots in sort order |
| `static RenderAttribRegistry *get_global_ptr()` | Lazily-initialized singleton accessor |

`typedef RenderAttribRegistry::SlotMask` is re-exported as `RenderState::SlotMask`.

## Usage

Application code doesn't call this directly; a subclass registers itself, e.g.:

```cpp
// inside ColorAttrib::init_type() or similar, once per process:
_slot = register_slot(get_class_type(), 100, make_default_ptr());
```

## See also

[RenderAttrib](RenderAttrib.md), [RenderState](RenderState.md).
