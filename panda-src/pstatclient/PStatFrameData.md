# PStatFrameData

**Source:** `panda/src/pstatclient/pStatFrameData.{h,I,cxx}`
**Inherits from:** none

Holds the raw timing and level data collected for a single thread during a
single frame: a sequence of start/stop events (by collector index and
timestamp) and a table of level values. This is the actual payload
transmitted to the PStats server — everything above it
([`PStatCollector`](PStatCollector.md), [`PStatThread`](PStatThread.md)) is
bookkeeping that eventually funnels into one of these per thread per frame.

## Behavior

**Time and level events are stored as parallel flat vectors of `{collector
index, value}` pairs**, not per-collector — `_time_data` holds every
start/stop event across every collector for the frame in the order they
occurred (`_value` is a timestamp; whether it's a start or a stop is encoded
in the sign — see `is_start()`), and `_level_data` holds one entry per
collector that had a level value set that frame. `sort_time()` does a
`std::stable_sort` over `_time_data` to guarantee monotonically increasing
timestamps before transmission — necessary because
[`PStatClient::start()`](PStatClient.md)'s `as_of`-timestamp overload allows
recording an event slightly out of real-time order.

**`write_datagram()` hand-rolls a fast binary path on little-endian/GCC
builds instead of going through the normal `Datagram` accessor calls.** On
`!WORDS_BIGENDIAN || __GNUC__`, it resizes the datagram's backing array once
and writes raw `uint16`/`float32` pairs directly through a pointer — "hand-
roll this, significantly more efficient for many data points" per the source
comment — with an explicit `__builtin_bswap16`/`32` byte-swap path for the
big-endian-but-GCC case. Big-endian non-GCC builds fall back to the normal
`Datagram::add_uint16()`/`add_float32()` calls. A frame with 65536 or more
time or level events is dropped entirely (logged, not sent) rather than
risking datagram corruption, since the wire format's original count field
was 16 bits.

**`read_datagram()` branches on protocol version for the count field
width.** `PStatClientVersion::is_at_least(3, 2)` selects a 32-bit
(`get_uint32()`) event count; older protocol versions (pre-3.2) use the
original 16-bit count. This is the one piece of this module where wire
compatibility with an older/newer server genuinely depends on negotiated
version — see [`PStatClientVersion.md`](PStatClientVersion.md).

## API reference

```cpp
class PStatFrameData {
public:
  bool is_time_empty() const;
  bool is_level_empty() const;
  bool is_empty() const;
  void clear();
  void swap(PStatFrameData &other);

  void add_start(int index, double time);
  void add_stop(int index, double time);
  void add_level(int index, double level);

  void sort_time();

  double get_start() const;
  double get_end() const;
  double get_net_time() const;

  size_t get_num_events() const;
  int get_time_collector(size_t n) const;
  bool is_start(size_t n) const;
  double get_time(size_t n) const;

  size_t get_num_levels() const;
  int get_level_collector(size_t n) const;
  double get_level(size_t n) const;

  bool write_datagram(Datagram &destination, PStatClient *client) const;
  void read_datagram(DatagramIterator &source, PStatClientVersion *version);
};
```

- `swap()` is used by `PStatClientImpl::new_frame()` to hand off a
  completed frame's buffer for transmission while immediately starting a
  fresh, empty buffer for the next frame — an O(1) exchange rather than a
  copy.
- `get_net_time()` is `get_end() - get_start()`, i.e. the total frame span
  from the first to the last recorded event.
- Not `PUBLISHED` — this is a pure C++ data-transport class, not exposed to
  Python.

## Usage

Not constructed directly by application code in the common case — populated
internally by [`PStatClient`](PStatClient.md)'s `start()`/`stop()`/
`set_level()` family and consumed by
[`PStatClientImpl`](PStatClientImpl.md)'s `transmit_frame_data()`. The
`PStatThread::add_frame()` lower-level API accepts a caller-assembled
`PStatFrameData` directly, for code that wants to construct frame data
out-of-band (e.g. importing recorded data rather than live-timing it).

## Related classes

- [`PStatClientImpl`](PStatClientImpl.md) — serializes and transmits this
  data every frame
- [`PStatThread`](PStatThread.md) — owns the in-progress `PStatFrameData` for
  its thread, via `PStatClient::InternalThread::_frame_data`
- [`PStatClientVersion`](PStatClientVersion.md) — determines the wire format
  used by `read_datagram()`
