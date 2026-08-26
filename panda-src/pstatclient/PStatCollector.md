# PStatCollector

**Source:** `panda/src/pstatclient/pStatCollector.{h,I}` (the `.cxx` exists but contains only the `#include` — every method is inline in the `.I`)
**Inherits from:** none

The lightweight value-type handle application code actually constructs to
measure one named category via PStats — either elapsed time (via
`start()`/`stop()`, typically through a [`PStatTimer`](PStatTimer.md)) or an
arbitrary numeric "level" (via `set_level()`/`add_level()` — triangle counts,
texture memory, anything the caller defines). A collector is cheap to copy
and typically kept as a `static` local or class member.

## Behavior

**Construction resolves (or creates) the collector by name through
[`PStatClient`](PStatClient.md), it doesn't own any state itself beyond a
client pointer, an index, and a small `_level` accumulator.** The
`(name, client)` constructor calls
`client->make_collector_with_relname(0, name)` (parented under `"Frame"` by
default, unless `name` contains colons); the `(parent, name)` constructor
does the same but resolves relative to `parent`'s index instead of 0.
Copying a `PStatCollector` copies the client pointer and index but always
resets the local `_level` to `0.0` — the accumulator is per-handle, not
shared across copies of "the same" collector.

**`is_valid()` distinguishes a real collector from a default-constructed
one** (`_client == nullptr`); the header is explicit that using an invalid
collector "will crash" — there's no runtime null-check on the hot
`start()`/`stop()` path.

**The no-thread-argument overloads resolve "the current thread" differently
depending on whether threading support is compiled in at all**: under
`HAVE_THREADS`, `start()`/`stop()`/`is_active()`/etc. call
`_client->get_current_thread()` (which itself skips the relatively expensive
`Thread::get_current_thread()` call entirely if the client isn't even
connected); without `HAVE_THREADS`, they hardcode thread index `0` and skip
the lookup altogether.

**`add_level()`/`sub_level()` (no-thread-argument overloads) don't touch the
client at all — they only accumulate into the handle's own `_level`.** The
comment in the header calls this out explicitly as "an optimization": the
data isn't sent to `PStatClient` until `flush_level()` runs, which happens
implicitly inside `set_level()`, `clear_level()`, and `get_level()`, or
explicitly via `add_level_now()`/`sub_level_now()`. This means calling
`add_level()` in a tight loop and only occasionally calling `get_level()` (or
letting the next `set_level()` flush it) avoids a client round-trip per
increment. The *thread-argument* overloads (`add_level(const PStatThread&,
...)`) skip this batching and write straight through to
`PStatClient::add_level()` every time.

**`get_index()` exposes the raw collector index** — used internally by
`PStatTimer` and by code (like the memory-usage reporting in
`PStatClient::main_tick()`) that needs to pass a collector reference through
a lower-level API that only deals in indices.

## API reference

```cpp
class PStatCollector {
PUBLISHED:
  explicit PStatCollector(const std::string &name, PStatClient *client = nullptr);
  explicit PStatCollector(const PStatCollector &parent, const std::string &name);

  PStatCollector(const PStatCollector &copy);
  void operator = (const PStatCollector &copy);

  bool is_valid() const;
  std::string get_name() const;
  std::string get_fullname() const;
  void output(std::ostream &out) const;

  bool is_active();
  bool is_started();
  void start();
  void stop();

  void clear_level();
  void set_level(double level);
  void add_level(double increment);
  void sub_level(double decrement);
  void add_level_now(double increment);
  void sub_level_now(double decrement);
  void flush_level();
  double get_level();

  void clear_thread_level();
  void set_thread_level(double level);
  void add_thread_level(double increment);
  void sub_thread_level(double decrement);
  double get_thread_level();

  // Thread-explicit overloads of the above (is_active/is_started/start/stop/
  // clear_level/set_level/add_level/sub_level/get_level), each taking a
  // `const PStatThread &thread` to operate on a thread other than the
  // current one.

  int get_index() const;
};
```

- `get_name()` is the local (rightmost) name segment; `get_fullname()`
  concatenates all ancestor names with `:`.
- `*_thread_level()` methods operate on the *current* thread explicitly
  (resolving it once, then routing through the thread-explicit overloads),
  as distinct from the plain `*_level()` methods which always target the
  main thread (index 0).
- `is_active()` reflects whether the collector's `PStatCollectorDef` is
  currently enabled *and* the client is connected; `is_started()` reflects
  whether `start()` has been called more times than `stop()` right now.

## Usage

```cpp
static PStatCollector my_pcollector("App:MySystem:DoWork");

my_pcollector.start();
// ... work ...
my_pcollector.stop();

// Or, more commonly, via PStatTimer for exception/early-return safety:
{
  PStatTimer timer(my_pcollector);
  // ... work ...
}

// Level tracking:
static PStatCollector triangle_pcollector("Vertices:Triangles");
triangle_pcollector.add_level(num_triangles);
```

## Related classes

- [`PStatTimer`](PStatTimer.md) — RAII wrapper that calls `start()`/`stop()`
  automatically
- [`PStatClient`](PStatClient.md) — owns the actual per-collector state this
  handle delegates to
- [`PStatCollectorDef`](PStatCollectorDef.md) — the metadata (color, sort,
  active flag) backing this collector's index
- [`PStatCollectorForward`](PStatCollectorForward.md) — a forward-reference
  wrapper for code that can't link against this class directly
