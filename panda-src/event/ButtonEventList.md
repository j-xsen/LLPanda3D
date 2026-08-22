# ButtonEventList

**Source:** `panda/src/event/buttonEventList.h` / `.I` / `.cxx`
**Inherits:** `ParamValueBase`

An ordered batch of [ButtonEvent](ButtonEvent.md)s that happened since the
last time the list was read — the form button input travels through Panda's
data graph in. Inheriting `ParamValueBase` means one of these can be wrapped
directly in an [EventParameter](EventParameter.md) (it's Bam-serializable too,
via `write_datagram`/`fillin`, for recording/playback).

## Behavior notes

- **Order is preserved and significant** — events are appended
  (`add_event`) and read back (`get_event(n)`) in chronological order; there's
  no reordering or dedup on insert.
- **`update_mods(ModifierButtons&)`** replays every event in the list through
  `ButtonEvent::update_mods()`, i.e. it's the batch form of tracking modifier
  key state (shift/ctrl/alt held) across a whole list at once.
- **`get_event(n)` is bounds-checked only in debug builds** — in an `NDEBUG`
  release build, out-of-range access is undefined; the `nassertr` guard (with
  a static empty-event fallback) only exists under `#ifndef NDEBUG`.
- **Bam serialization is registered at module init** (`config_event.cxx` calls
  `ButtonEventList::register_with_read_factory()`), so these can appear inside
  `.bam` files (e.g. recorded input sessions via `MouseRecorder`) —
  `fillin()` is deliberately public rather than protected specifically so
  `MouseRecorder` can call it mid-datagram.

## API

| Signature | Notes |
|---|---|
| `void add_event(ButtonEvent event)` | Appends |
| `int get_num_events() const` / `const ButtonEvent &get_event(int n) const` | `MAKE_SEQ`'d as `events` in Python |
| `void clear()` | |
| `void add_events(const ButtonEventList &other)` | Appends another list's events onto this one |
| `void update_mods(ModifierButtons &mods) const` | Batch-replay through every event |
| `void output(std::ostream&) const` / `void write(std::ostream&, int indent_level = 0) const` | One-line vs. multi-line dump |

## Usage

```cpp
// Typically obtained from the data graph (e.g. a ButtonThrower/InputDevice node's
// output), not constructed by hand:
const ButtonEventList *events = ...;
for (int i = 0; i < events->get_num_events(); ++i) {
  const ButtonEvent &be = events->get_event(i);
  // ...
}
```

## See also

[ButtonEvent.md](ButtonEvent.md) · [PointerEventList.md](PointerEventList.md) ·
[EventParameter.md](EventParameter.md) (how this gets attached to an `Event`) ·
[README.md](README.md)
