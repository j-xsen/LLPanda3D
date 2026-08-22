# EventHandler

**Source:** `panda/src/event/eventHandler.h` / `.I` / `.cxx`
**Inherits:** `TypedObject`

Monitors events "from the C++ side" — maintains a table of event-name → hook
(plain C function pointer, optionally with a `void*` userdata payload) and
calls the right hooks when a matching event is dispatched. This is how C++
code (not Python's messenger) subscribes to named events.

## Behavior notes

- **Two independent hook tables exist**, both keyed by event name:
  `EventFunction*` hooks (`void(const Event*)`, no extra data — a `pset`, so
  the same function pointer can only be registered once per event name) and
  `EventCallbackFunction*` hooks (`void(const Event*, void*)`, paired with a
  userdata pointer — the pair `(function, data)` is the dedup key, so the same
  function with *different* userdata can be registered multiple times on one
  event). Both fire on every `dispatch_event()` call for a matching name.
- **`process_events()` just loops `dispatch_event(queue.dequeue_event())`
  until the queue is empty** — it does not snapshot the queue first, so an
  event thrown *during* dispatch of another event will also be processed in
  the same `process_events()` call (not deferred to next frame).
- **`dispatch_event()` copies the function set before iterating** ("`Functions
  copy_functions = (*hi).second;`") specifically so a hook is allowed to
  add/remove hooks (including itself) during its own callback without
  invalidating the iteration.
- **Futures integrate transparently.** `get_future(event_name)` returns an
  `AsyncFuture` that `dispatch_event()` automatically resolves (via
  `set_result()`) the next time that event name fires — the entry is then
  erased from the futures table (one-shot; call `get_future()` again for the
  next occurrence). If you call `get_future()` again before the event fires,
  you get back the *same* pending future rather than a duplicate, unless the
  previous one was explicitly cancelled.
- **`remove_hooks(event_name)` clears both tables for that name**;
  `remove_hooks_with(data)` scans every callback-hook entry and removes any
  whose userdata pointer matches, regardless of event name or function —
  useful for "detach everything belonging to this object" cleanup.
- **The global handler ignores its `queue` parameter.** `get_global_event_handler(queue)`
  keeps the parameter only for backward compatibility; it always binds to
  `EventQueue::get_global_event_queue()` internally the first time it's
  created, regardless of what you pass.

## EventReceiver

**Source:** `panda/src/event/eventReceiver.h` / `.cxx`

A near-empty abstract tag base class:

```cpp
class EventReceiver {
public:
  static TypeHandle get_class_type();
};
```

Anything that might be the optional `receiver` field on an [Event](Event.md)
should inherit from this — it exists purely to give that field a common,
identifiable base type; it defines no virtual interface of its own.

## API

| Signature | Notes |
|---|---|
| `explicit EventHandler(EventQueue *ev_queue)` | Binds to a specific queue |
| `static EventHandler *get_global_event_handler(EventQueue *queue = nullptr)` | Lazily created singleton bound to the global queue; `queue` param is ignored |
| `void process_events()` | Drains and dispatches every event currently on the bound queue |
| `virtual void dispatch_event(const Event *event)` | Dispatches one event immediately, bypassing the queue |
| `AsyncFuture *get_future(const std::string &event_name)` | One-shot future resolved on next occurrence of the named event |
| `bool add_hook(const std::string &event_name, EventFunction *function)` | Returns false if already registered |
| `bool add_hook(const std::string &event_name, EventCallbackFunction *function, void *data)` | `(function, data)` pair is the identity |
| `bool has_hook(event_name) const` / `has_hook(event_name, function) const` / `has_hook(event_name, function, data) const` | Overloads for each hook kind |
| `bool remove_hook(event_name, function)` / `remove_hook(event_name, function, data)` | |
| `bool remove_hooks(const std::string &event_name)` | Clears both tables for one event name |
| `bool remove_hooks_with(void *data)` | Removes every callback hook using this userdata, any event |
| `void remove_all_hooks()` | Nukes both tables entirely |
| `void write(std::ostream&) const` | Debug dump of every registered hook |

## Usage

```cpp
EventHandler *eh = EventHandler::get_global_event_handler();

void on_escape(const Event *event) { /* ... */ }
eh->add_hook("escape", on_escape);

// Per-frame (normally done by the application framework):
eh->process_events();

// Await an event as a future, e.g. from a coroutine-style task:
PT(AsyncFuture) fut = eh->get_future("level-loaded");
```

## See also

[Event.md](Event.md) · [EventQueue.md](EventQueue.md) · [throw_event.md](throw_event.md)
(`throw_event_directly()` calls `dispatch_event()` on a handler, bypassing the
queue) · [AsyncFuture.md](AsyncFuture.md) · [README.md](README.md)
