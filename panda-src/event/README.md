# Event — Panda3D's Event Dispatch & Async Task System

**Source:** `panda/src/event/` · Library: `libp3event` · Notify categories: `event`, `task`

This module contains two related but independent subsystems that ship in the
same library:

1. **Event dispatch** — Panda's global, string-named publish/subscribe system.
   Anything anywhere (C++ or Python, any thread) can throw a named event with
   optional parameters; a queue holds them until a handler processes them by
   calling registered C++ function-pointer hooks. This is the same mechanism
   [PGUI](../pgui/README.md) uses for `press-`, `click-`, etc. — `PGItem`'s
   `throw_event()` calls are calls into *this* module.
2. **Async tasks** — Panda's per-frame task scheduler, the C++ engine behind
   Python's `taskMgr`. Tasks are added to an `AsyncTaskManager`, which runs
   each one once per "epoch" (frame) via one or more `AsyncTaskChain`s,
   optionally on background threads.

The two subsystems intersect at `AsyncFuture`: `EventHandler::get_future()`
returns a future that resolves the next time a named event fires, and a task
can await it (`DS_await`) to be woken by an event rather than polled.

## Class map

**Event dispatch:**
```
TypedReferenceCount
└── Event                       (Event.md)         — a named event + parameter list

EventParameter                  (EventParameter.md) — typed payload wrapper (value type)

EventQueue                      (EventQueue.md)     — FIFO of pending Events; one global instance
EventHandler (: TypedObject)    (EventHandler.md)   — drains a queue, dispatches to registered hooks
EventReceiver                   (EventHandler.md#eventreceiver) — tag base class, optional Event owner

ParamValueBase
├── ButtonEventList              (ButtonEventList.md) — recent keyboard/mouse button transitions
└── PointerEventList             (PointerEventList.md) — recent pointer/mouse-motion samples
ButtonEvent                      (ButtonEvent.md)    — one button up/down/keystroke/candidate event (value type)
PointerEvent                     (PointerEvent.md)   — one pointer-motion sample (value type)

throw_event() / throw_event_directly()  (throw_event.md) — free functions, the normal way to fire an event
```

**Async tasks:**
```
TypedReferenceCount
└── AsyncFuture                  (AsyncFuture.md)    — thread-safe promise/future; base of AsyncTask
    └── AsyncTask (: Namable)    (AsyncTask.md)      — one schedulable unit of work
        ├── GenericAsyncTask      (AsyncTask.md#genericasynctask)  — wraps a C function pointer, no subclassing needed
        ├── AsyncTaskPause        (AsyncTask.md#asynctaskpause)    — sleeps N seconds then finishes; for sequences
        └── AsyncTaskSequence (: AsyncTaskCollection) (AsyncTaskSequence.md) — runs a list of tasks one per epoch
    └── AsyncGatheringFuture      (AsyncFuture.md#asyncgatheringfuture) — resolves when N other futures all resolve

AsyncTaskCollection               (AsyncTaskCollection.md) — plain list of AsyncTask pointers
AsyncTaskChain (: TypedReferenceCount, Namable)  (AsyncTaskChain.md)   — one independently-scheduled queue + thread pool
AsyncTaskManager (: TypedReferenceCount, Namable) (AsyncTaskManager.md) — owns a set of named AsyncTaskChains; one global instance
```

Not documented here (out of scope for this C++ reference):
- **`PythonTask`** (`pythonTask.h/.cxx`) — `AsyncTask` subclass that wraps a
  Python generator/callable; Python-interop only, not part of the C++-facing
  API surface.
- **`asyncFuture_ext.h/.cxx`** — `AsyncFuture`'s Python `EXTENSION(...)`
  method implementations (`__await__`, `result()`, etc.); also Python-only.
- **`test_task.cxx`** — a standalone test/demo program, not library code.
- **`config_event`** — module config/init boilerplate (registers types, no
  runtime API of its own); its two notify categories (`event`, `task`) are
  noted above.
- **`pt_Event`** (`pt_Event.h`) — only the explicit template-instantiation
  boilerplate for `PT(Event)`/`CPT(Event)` (aliased as `PT_Event`/`CPT_Event`);
  no API beyond the typedefs, mentioned in [Event.md](Event.md).

## Core concepts

**Events are fire-and-forget strings, not typed messages.** `throw_event("foo")`
queues an `Event` object; nothing about the call site knows or cares whether
anyone is listening. Zero, one, or many hooks may be registered for any given
name, and hooks are plain C function pointers (`void (*)(const Event *)`) or
callback-with-userdata pointers — there is no C++ virtual-dispatch/observer
class hierarchy here (contrast with [PGUI's Notify interfaces](../pgui/PGItem.md#notify-interface--pgitemnotify),
which *are* virtual callbacks and are a different, narrower mechanism specific
to `PGItem`).

**Queue and handler are separate, and both have a lazily-created global
singleton.** `EventQueue::get_global_event_queue()` and
`EventHandler::get_global_event_handler()` are what most game code uses;
`throw_event()` always queues onto the global queue, and
`EventHandler::process_events()` (called once per frame by the application
framework) drains it and dispatches to whatever hooks are registered on the
global handler. Applications may construct their own `EventQueue`/`EventHandler`
pairs for isolated event domains, though this is uncommon.

**`EventParameter` is a type-erased, refcounted value box.** It wraps an int,
double, `std::string`, `std::wstring`, or any `TypedWritableReferenceCount` /
`TypedReferenceCount` pointer behind one uniform type, so `Event::add_parameter()`
and `throw_event(name, p1, p2, ...)` can accept up to 4 parameters of mixed
types. Consumers check `is_int()`/`is_double()`/`is_string()`/... before
calling the matching `get_*_value()`.

**Tasks run once per "epoch" (frame), not continuously.** An `AsyncTask`'s
`do_task()` is called, does a bounded amount of work, and returns a
`DoneStatus` telling the manager what to do next time: `DS_done` (remove),
`DS_cont` (run again next epoch), `DS_again` (restart from scratch, resetting
elapsed-time bookkeeping), `DS_pickup` (run again *this* frame if there's
budget left), `DS_pause`/`DS_exit` (sequence-specific stop/pause), `DS_interrupt`
(abort the whole manager's poll this frame, retry next epoch), `DS_await`
(suspend until another `AsyncFuture` completes). This is the same status enum
Python's `task.cont`/`task.done`/etc. map to.

**Sort vs. priority are different axes.** `sort` partitions tasks into
strictly-ordered groups — all sort-0 tasks finish before any sort-1 task
starts, ever, even with multiple threads. `priority` only orders tasks
*within* the same sort value, and (unless `timeslice_priority` is enabled)
does not change how often a task runs — every task still runs exactly once
per epoch by default. See [AsyncTaskChain.md](AsyncTaskChain.md).

**A task belongs to a named chain, and chains belong to a manager.** Every
task manager starts with one implicit chain named `"default"`. Assigning a
task to a different chain name via `set_task_chain()` auto-creates that chain
(with a logged warning) if it doesn't exist yet. Only tasks on *different*
chains can truly run in parallel across threads; see
[AsyncTaskManager.md](AsyncTaskManager.md) and [AsyncTaskChain.md](AsyncTaskChain.md).

**`AsyncFuture` is the common base of a task and a plain promise.** Both
support `done()`, `cancel()`, `wait()`, and a `done_event` that fires when
they complete. This is why `EventHandler::get_future(event_name)` — a future
that resolves the next time that named event throws — can be awaited by a
task exactly like another task's completion can.

## File index

| Topic | Purpose |
|---|---|
| [Event.md](Event.md) | A named event + parameter list; what gets queued and dispatched |
| [EventParameter.md](EventParameter.md) | Type-erased value box carried by events |
| [EventQueue.md](EventQueue.md) | FIFO of pending events; the global queue `throw_event()` writes to |
| [EventHandler.md](EventHandler.md) | Drains a queue, dispatches to registered C-function hooks (+ `EventReceiver`) |
| [throw_event.md](throw_event.md) | Free functions — the normal way application code fires an event |
| [ButtonEvent.md](ButtonEvent.md) | One keyboard/mouse button transition or keystroke (value type) |
| [ButtonEventList.md](ButtonEventList.md) | A batch of `ButtonEvent`s, as carried through the data graph |
| [PointerEvent.md](PointerEvent.md) | One pointer-motion sample (value type) |
| [PointerEventList.md](PointerEventList.md) | A batch of `PointerEvent`s, with gesture-analysis helpers |
| [AsyncFuture.md](AsyncFuture.md) | Thread-safe promise/future; base of `AsyncTask` (+ `AsyncGatheringFuture`) |
| [AsyncTask.md](AsyncTask.md) | One schedulable unit of work (+ `GenericAsyncTask`, `AsyncTaskPause`) |
| [AsyncTaskSequence.md](AsyncTaskSequence.md) | A task that runs a list of sub-tasks one per epoch |
| [AsyncTaskCollection.md](AsyncTaskCollection.md) | Plain list container for `AsyncTask` pointers |
| [AsyncTaskChain.md](AsyncTaskChain.md) | One independently-scheduled task queue + optional thread pool |
| [AsyncTaskManager.md](AsyncTaskManager.md) | Owns named `AsyncTaskChain`s; the global task scheduler |

## Status

event — done (2026-08-22). See [../../README.md](../../README.md) for the
overall index across `panda/src/*` modules.
