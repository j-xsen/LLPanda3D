# throw_event / throw_event_directly

**Source:** `panda/src/event/throw_event.h` / `.I`

Free-function convenience wrappers — the normal way application/engine code
fires an event. Not a class; just inline functions.

## Behavior notes

- **`throw_event(...)` always goes through the global queue**
  (`EventQueue::get_global_event_queue()->queue_event(...)`) — the event is
  *not* dispatched synchronously; it sits queued until something calls
  `EventHandler::process_events()` (normally once per frame).
- **`throw_event_directly(handler, ...)` skips the queue entirely** and calls
  `handler.dispatch_event(...)` immediately, synchronously, on the calling
  thread. This is reserved for cases specifically needing synchronous
  delivery to a *particular* `EventHandler` (not the global one) — it takes
  an `EventHandler&`, not just an event name, so a handler instance must
  already be in hand.
- **Both families overload on 0–4 `EventParameter` arguments** — there's no
  variadic/vector form; more than 4 parameters requires constructing an
  [Event](Event.md) directly and calling `add_parameter()` repeatedly, then
  passing it to the `CPT_Event` overload.
- **The `CPT_Event` overload of `throw_event`/`throw_event_directly`** accepts
  a pre-built `Event` (as `PT_Event`/`CPT_Event`) — used to reuse a
  constructed event or one already built (e.g. `AsyncFuture::notify_done()`
  builds an `Event` and passes it this way).

## API

```cpp
void throw_event(const CPT_Event &event);
void throw_event(const std::string &event_name);
void throw_event(const std::string &event_name, const EventParameter &p1);
void throw_event(const std::string &event_name, const EventParameter &p1, const EventParameter &p2);
void throw_event(const std::string &event_name, const EventParameter &p1, const EventParameter &p2, const EventParameter &p3);
void throw_event(const std::string &event_name, const EventParameter &p1, const EventParameter &p2, const EventParameter &p3, const EventParameter &p4);

void throw_event_directly(EventHandler &handler, const CPT_Event &event);
void throw_event_directly(EventHandler &handler, const std::string &event_name);
void throw_event_directly(EventHandler &handler, const std::string &event_name, const EventParameter &p1);
void throw_event_directly(EventHandler &handler, const std::string &event_name, const EventParameter &p1, const EventParameter &p2);
void throw_event_directly(EventHandler &handler, const std::string &event_name, const EventParameter &p1, const EventParameter &p2, const EventParameter &p3);
```

## Usage

```cpp
#include "throw_event.h"

throw_event("player-scored", EventParameter(player_id), EventParameter(points));

// Rare: bypass the queue for a specific handler, synchronous delivery.
throw_event_directly(*my_handler, "urgent-shutdown");
```

## See also

[EventQueue.md](EventQueue.md) (where `throw_event` writes to) ·
[EventHandler.md](EventHandler.md) (`dispatch_event`, called directly by
`throw_event_directly`) · [Event.md](Event.md) · [EventParameter.md](EventParameter.md) ·
[README.md](README.md)
