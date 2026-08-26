# PStatClientImpl

**Source:** `panda/src/pstatclient/pStatClientImpl.{h,I,cxx}` (exists only when `DO_PSTATS` is defined — no stub)
**Inherits from:** `ConnectionManager` (from `net`, undocumented here)

The actual network/protocol implementation behind [`PStatClient`](PStatClient.md).
Split out from `PStatClient` specifically so the global `PStatClient` singleton
can be constructed at static-init time without touching any config
variables — `PStatClientImpl` (and the socket setup that depends on
`pstats-*` config vars) is only created lazily, the first time
`PStatClient::get_impl()` is called, which in practice means the first time
`connect()` is called.

## Behavior

**Construction reads the TCP/UDP send ratio out of `pstats-tcp-ratio` once,
via a sine/cosine split rather than a plain percentage.** `pstats-tcp-ratio`
(0.0–1.0) is converted with `csincos(ratio * π/2, &_udp_count_factor,
&_tcp_count_factor)`; each `transmit_frame_data()` call then compares
`_udp_count * _udp_count_factor` against `_tcp_count * _tcp_count_factor`
and increments whichever counter's weighted value is currently lower — an
interleaving scheme that approximates the target ratio over many packets
without needing a random number generator. A ratio of exactly `0.0` or
`1.0` short-circuits to picking UDP or TCP unconditionally.

**`client_connect()` opens the TCP connection synchronously (5-second
timeout) and fires off `send_hello()` immediately**, before any UDP port is
known. `_got_udp_port` stays false — and therefore no thread is marked
`_is_active` and no frame data is recorded — until the server's `T_hello`
reply arrives on the TCP channel (see `handle_server_control_message()`)
carrying the UDP port to send data to. `set_collect_tcp(false)` is set on
the TCP connection explicitly, because control messages need to go out
immediately rather than batching.

**`new_frame()` is the actual per-frame pipeline**: it stops collector 0
(`"Frame"`) at the current time to close out the *previous* frame's
`PStatFrameData`, appends every collector's buffered level data
(`PerThreadData::_level` values for anything with `_has_level` set) into
that same buffer, swaps the completed buffer out for transmission, then
immediately starts collector 0 again for the *new* frame before handing the
old buffer to `transmit_frame_data()`. The client's own transmission time is
itself measured — bracketed in the `"*:PStats"` collector
(`PStatClient::_pstats_pcollector`) — so PStats overhead shows up in its own
graph.

**`transmit_frame_data()` rate-limits per thread via `_next_packet`,
independent of the TCP/UDP choice.** A packet is dropped entirely (not
queued) if not enough time has passed since the last one for that thread,
based on `1.0 / _max_rate` seconds; if the datagram turns out too large for
UDP, it's sent via TCP instead and `packet_delay` is further multiplied by
however many UDP-sized chunks it would have taken, so oversized frames
throttle themselves harder rather than flooding the TCP channel.

**`transmit_control_data()` runs once per frame (main thread only,
`thread_index == 0`) as a `while (data_available())` drain**, so any number
of queued server messages are processed in one call, followed by
`report_new_collectors()`/`report_new_threads()` to push any collectors or
threads created since the last tick. Both report functions track a running
"reported so far" count (`_collectors_reported`/`_threads_reported`) and
resume from there — `report_new_collectors()` additionally caps each
`T_define_collectors` message at 700 collectors, "empirically determined"
to stay under the 64K datagram size limit.

**Losing the UDP connection permanently downgrades to TCP-only for that
session.** `connection_reset()` (a `ConnectionManager` virtual override)
distinguishes TCP loss (triggers a full `client_disconnect()`) from UDP loss
(just flips `_tcp_count_factor`/`_udp_count_factor` to force all future
packets over TCP) — there's no attempt to re-establish UDP later.

## API reference

```cpp
class PStatClientImpl : public ConnectionManager {
public:
  PStatClientImpl(PStatClient *client);
  ~PStatClientImpl();

  void set_client_name(const std::string &name);
  std::string get_client_name() const;
  void set_max_rate(double rate);
  double get_max_rate() const;

  double get_real_time() const;

  void client_main_tick();
  bool client_connect(std::string hostname, int port);
  void client_disconnect();
  bool client_is_connected() const;

  void client_resume_after_pause();

  void new_frame(int thread_index);
  void add_frame(int thread_index, const PStatFrameData &frame_data);
};
```

- `get_real_time()` uses its own `TrueClock` plus a `_delta` offset (adjusted
  by `client_resume_after_pause()`), independent of the global `ClockObject`.
- `client_connect()` falls back to `pstats_host`/`pstats_port` config values
  when passed an empty hostname / negative port.
- `add_frame()` is the lower-level alternative to `new_frame()` used when the
  caller already has a fully-assembled `PStatFrameData` to hand over (see
  [`PStatThread::add_frame()`](PStatThread.md)).

## Usage

Never constructed or referenced directly by application code — it's an
internal implementation detail of [`PStatClient`](PStatClient.md), created
on demand by `PStatClient::make_impl()`.

## Related classes

- [`PStatClient`](PStatClient.md) — owns exactly one `PStatClientImpl`,
  created lazily
- [`PStatClientControlMessage`](PStatClientControlMessage.md) /
  [`PStatServerControlMessage`](PStatServerControlMessage.md) — the TCP
  control protocol this class encodes/decodes
- [`PStatFrameData`](PStatFrameData.md) — the per-thread data this class
  serializes and sends
