# PGMouseWatcherParameter

**Source:** `panda/src/pgui/pgMouseWatcherParameter.h` / `.I` / `.cxx`
**Inherits:** `TypedWritableReferenceCount`, `MouseWatcherParameter`

The event-payload type attached to nearly every PG event
(`throw_event(event, EventParameter(ep))`). It adds nothing behaviorally over
plain `MouseWatcherParameter` — same fields (button, mouse position, keycode,
IME candidate info, etc.) — it exists purely so the parameter can be
reference-counted and attached to a Panda `Event` as an `EventParameter`
(`MouseWatcherParameter` alone isn't a `TypedWritableReferenceCount`).

```cpp
class PGMouseWatcherParameter
  : public TypedWritableReferenceCount, public MouseWatcherParameter {
public:
  PGMouseWatcherParameter();
  PGMouseWatcherParameter(const MouseWatcherParameter &copy);
  void operator = (const MouseWatcherParameter &copy);
  void output(std::ostream &out) const;
};
```

**Inheritance order is load-bearing:** `TypedWritableReferenceCount` is listed
first even though `MouseWatcherParameter` is conceptually primary — MSVC++
places the *first* listed base at the front of the object layout, and
`interrogate` (Panda's C++→Python binding generator) assumes the same.

In an event handler, retrieve it from an `Event` the normal Panda way (cast
the `EventParameter`'s pointer back to `PGMouseWatcherParameter`) to read
`get_button()`, `get_mouse()`, `get_keycode()`, `is_outside()`,
`get_candidate_string()`, etc. — all inherited from `MouseWatcherParameter`,
not redeclared here.

## See also

[PGItem.md](PGItem.md#events) (events that carry this type) · [README.md](README.md)
