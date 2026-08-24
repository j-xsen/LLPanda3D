# Event

**Source:** `panda/src/event/event.h` / `.I` / `.cxx` (+ `pt_Event.h`)
**Inherits:** `TypedReferenceCount`

A named event plus zero or more [EventParameter](EventParameter.md)s. This is
what [throw_event()](throw_event.md) constructs and queues, and what
[EventHandler](EventHandler.md) hands to a registered hook function. Application
code almost never constructs an `Event` directly — `throw_event()` is used
instead — but hook functions receive a `const Event *` and read it.

## Behavior notes

- **`PT_Event`/`CPT_Event`** (from `pt_Event.h`) are just
  `PointerTo<Event>`/`ConstPointerTo<Event>` typedefs, defined in their own
  header purely so the template instantiation can be explicitly exported from
  the DLL. `EventQueue`/`EventHandler`/`throw_event` all traffic in
  `CPT_Event`, not raw `Event*`.
- **The optional `receiver`** (`EventReceiver*`) is a legacy/secondary
  targeting mechanism — an event can name a specific `EventReceiver` object it
  concerns, separate from its parameter list. It is not required and most code
  never touches `get_receiver()`/`set_receiver()`.
- **An event with an empty name is silently dropped**, not by `Event` itself,
  but by [EventQueue::queue_event()](EventQueue.md) refusing to enqueue it.
- **No longer inherits `Namable`** (per the header comment) — it manually
  duplicates a name field/getter/setter instead, because inheriting `Namable`
  made `get_name()` too expensive to call from Python. Functionally identical
  to a plain name field either way from C++.

## API

| Signature | Notes |
|---|---|
| `Event(const std::string &event_name, EventReceiver *receiver = nullptr)` | |
| `void set_name(const std::string&)` / `clear_name()` / `bool has_name() const` / `const std::string &get_name() const` | |
| `void add_parameter(const EventParameter &obj)` | Appends; order is preserved and meaningful (positional) |
| `int get_num_parameters() const` / `EventParameter get_parameter(int n) const` | `MAKE_SEQ`'d as `parameters` in Python |
| `bool has_receiver() const` / `EventReceiver *get_receiver() const` / `void set_receiver(EventReceiver*)` / `clear_receiver()` | See "Behavior notes" |
| `void output(std::ostream&) const` | `name(param1, param2, ...)` format |

## Usage

```cpp
// Normal path — don't construct Event directly:
throw_event("player-died", EventParameter(player_id));

// Reading one in a hook:
void on_player_died(const Event *event) {
  EventParameter p = event->get_parameter(0);
  if (p.is_int()) {
    int player_id = p.get_int_value();
  }
}
```

## See also

[EventParameter.md](EventParameter.md) · [throw_event.md](throw_event.md) ·
[EventQueue.md](EventQueue.md) · [EventHandler.md](EventHandler.md) ·
[README.md](README.md)
