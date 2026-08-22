# EventQueue

**Source:** `panda/src/event/eventQueue.h` / `.I` / `.cxx`
**Inherits:** none

A thread-safe FIFO of pending [Event](Event.md)s. [throw_event()](throw_event.md)
pushes onto the global queue; [EventHandler::process_events()](EventHandler.md)
pops everything off and dispatches it. You'll rarely touch an `EventQueue`
directly beyond `get_global_event_queue()` — it exists as its own class mainly
so alternate/isolated event pipelines are possible.

## Behavior notes

- **`get_global_event_queue()` lazily creates the one global instance** on
  first call — there's no explicit init step required.
- **Queuing an event with an empty name is silently dropped** — `queue_event()`
  checks `event->get_name().empty()` and returns immediately without enqueuing
  if so, no error.
- **`is_queue_full()` always returns `false`** — it's marked deprecated in the
  header; the queue is unbounded (a `pdeque`), so there is no "full" state.
- **Locked with a `LightMutex`** — safe to call `queue_event()` from any
  thread at any time, which is the whole point (events can be thrown from
  background threads).

## API

| Signature | Notes |
|---|---|
| `void queue_event(CPT_Event event)` | No-op if the event's name is empty |
| `void clear()` | Discards all pending events without dispatching them |
| `bool is_queue_empty() const` | |
| `bool is_queue_full() const` | Deprecated; always `false` |
| `CPT_Event dequeue_event()` | Pops the oldest event; undefined if the queue is empty (caller should check `is_queue_empty()` first) |
| `static EventQueue *get_global_event_queue()` | Lazily created singleton |

## Usage

```cpp
// Draining manually (normally EventHandler::process_events() does this for you):
EventQueue *q = EventQueue::get_global_event_queue();
while (!q->is_queue_empty()) {
  CPT_Event event = q->dequeue_event();
  // handle event
}
```

## See also

[Event.md](Event.md) · [EventHandler.md](EventHandler.md) (the normal consumer
of this queue) · [throw_event.md](throw_event.md) (the normal producer) ·
[README.md](README.md)
