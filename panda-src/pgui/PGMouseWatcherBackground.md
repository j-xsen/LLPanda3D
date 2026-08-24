# PGMouseWatcherBackground

**Source:** `panda/src/pgui/pgMouseWatcherBackground.h` / `.cxx`
**Inherits:** `MouseWatcherRegion`

A special region with no rectangle, never active for mouse purposes, that
exists solely to route every keypress in the application to whichever
[PGItem](PGItem.md)s have `set_background_focus(true)` — see PGItem's
"Behavior notes" for what background focus means from the widget side.

```cpp
class PGMouseWatcherBackground : public MouseWatcherRegion {
PUBLISHED:
  PGMouseWatcherBackground();   // set_active(false), set_keyboard(true)
public:
  // press/release/keystroke/candidate each call the corresponding
  // PGItem::background_* static dispatcher (see PGItem.md).
};
```

This is constructed and registered with a `MouseWatcher` (e.g.
`watcher->add_region(new PGMouseWatcherBackground)`) only when wiring up
a `MouseWatcher` by hand outside the normal `ShowBase`/`PGTop` setup; Panda's
default application setup already has an equivalent path so background-focus
`PGItem`s work out of the box.

## See also

[PGItem.md](PGItem.md) (background focus, `background_press`/`background_release`/
etc. static dispatchers) · [README.md](README.md)
