# PStats Client — Profiling Data Collection and Transport

**Source:** `panda/src/pstatclient/` · Library: `libp3pstatclient` · Notify category: `pstats`

`pstatclient` is the engine-side half of Panda3D's profiler: it defines named,
hierarchical `PStatCollector`s that time (or count) sections of engine and
application code, and streams the results over the network to an external
PStats server/GUI for live graphing. Almost every other subsystem creates its
own collectors and wraps hot code paths in a [`PStatTimer`](PStatTimer.md) —
see `../cull/README.md`'s "PStats" note and `../pipeline/Thread.md`'s
`PStatsCallback` section for two concrete consumers already documented
elsewhere. This module has no dependency on any rendering subsystem itself;
it only depends on `net` (for the socket layer, undocumented here), `express`
(for `PStatCollectorForwardBase`, undocumented here), and `pipeline`/`putil`.

This directory documents the public C++ API of every class in
`panda/src/pstatclient`, for use without re-reading the engine source.

## Class map

```
PStatClient                       (PStatClient.md)
└── PStatClientImpl                (PStatClientImpl.md)      — internal TCP/UDP protocol impl, created lazily
                                                                 the first time PStatClient needs it

PStatCollector                     (PStatCollector.md)       — lightweight handle app code constructs to
                                                                 time/level-track a section of code
PStatCollectorDef                  (PStatCollectorDef.md)    — name/color/units/sort metadata backing a collector
PStatCollectorForward : PStatCollectorForwardBase
                                    (PStatCollectorForward.md) — cheap forward-reference handle, for classes
                                                                   defined before this module can be linked against
PStatThread                        (PStatThread.md)          — per-Thread handle; independent timing/level state
PStatTimer                         (PStatTimer.md)           — RAII start()/stop() wrapper around a PStatCollector
                                                                 (header-only, no .cxx)
PStatFrameData                     (PStatFrameData.md)       — raw per-frame start/stop/level event buffer
PStatClientVersion : ReferenceCount
                                    (PStatClientVersion.md)  — protocol version negotiation
PStatClientControlMessage          (PStatClientControlMessage.md) — client→server TCP control message
PStatServerControlMessage          (PStatServerControlMessage.md) — server→client TCP control message
```

**Excluded from these docs:**
- `pStatClient_ext.{h,I,cxx}` — Python-only glue for `PStatClient` (`HAVE_PYTHON && DO_PSTATS`), folded into
  [`PStatClient.md`](PStatClient.md)'s API reference as a short note rather than given its own file.
- `pStatProperties.{h,cxx}` — no class; two free functions (protocol version numbers, noted in
  [`PStatClientVersion.md`](PStatClientVersion.md)) plus the static predefined-collector-properties table
  described in "Core concepts" below.
- `config_pstatclient.{h,cxx}` — library init + config vars, covered below. `config_pstats.h` is a stub left
  behind for the 1.10.x cycle that just `#error`s pointing at the renamed header; nothing to document.
- `p3pstatclient_composite{1,2}.cxx` — unity-build wrapper files.
- `test_client.cxx` — standalone test program.

## Core concepts

**`DO_PSTATS` compiles the entire module in or out.** Every public class here
(`PStatClient`, `PStatCollector`, `PStatThread`, `PStatTimer`,
`PStatCollectorForward`) has two complete definitions in the same header,
gated by `#ifdef DO_PSTATS`/`#else` — the "off" branch keeps the same public
signatures but makes every method a no-op inline stub (`start()`/`stop()` do
nothing, `get_level()` always returns `0.0`), so calling code never needs its
own `#ifdef`s. `PStatClientImpl` doesn't exist at all — not even a stub —
when `DO_PSTATS` is undefined; `PStatClient` simply never creates one.

**Collector names form a hierarchy via `:` separators.**
`PStatClient::make_collector_with_relname()` walks a name like `"Cull:Sort"`
one path segment at a time, creating (or reusing) a parent collector for each
segment; an unqualified name is parented under collector 0, `"Frame"`. A
quirk worth knowing: naming a child the same as its own parent collapses to
the parent — `PStatCollector(cull_collector, "Cull")` under a collector
already named `"Cull"` just returns `"Cull"` again, not a redundant
`"Cull:Cull"`.

**Time collectors use a nesting counter, not a boolean, so recursive
start()/stop() calls are safe.** `PStatClient::start()`/`stop()` only emit a
real timing datapoint on the collector's `_nested_count` going 0→1 (start)
or 1→0 (stop) for a given collector+thread pair; every other nested
call/return just bumps the counter. `PStatCollector::is_started()` reflects
`_nested_count != 0`, not a literal "was start() the most recent call."

**Level collectors batch writes through a local accumulator before touching
the client.** `PStatCollector::add_level()`/`sub_level()` (main-thread
overloads) only accumulate into the handle's own `double _level` member —
they never call into `PStatClient` directly. The accumulated delta is only
flushed to the client's real per-thread level state by `flush_level()`,
which runs implicitly inside `set_level()`, `clear_level()`, and
`get_level()`, or explicitly via `add_level_now()`/`sub_level_now()`. This
avoids a client-lock round trip on every single increment in a tight loop.
Thread-specific overloads (`add_level(const PStatThread&, ...)`) skip this
optimization and write straight through to `PStatClient::add_level()`.

**Every `Thread` PStats has seen gets independent per-thread timing/level
state.** `PStatThread` maps 1:1 to a `Thread` via
`Thread::get_pstats_index()`/`set_pstats_index()`, lazily assigned the first
time that thread is encountered (`PStatClient::do_make_thread()`). Each
thread also gets registered as a `Thread::PStatsCallback` target — `Thread`
calls `PStatClient::activate_hook()`/`deactivate_hook()` on context switches
(see `../pipeline/Thread.md`), which the client uses to bracket a
`"Wait:Thread block"` collector around time the thread spends swapped out —
most useful under `SIMPLE_THREADS`, where multiple logical threads share one
OS thread. Each collector's per-thread nested-count/level data is guarded by
that *thread's own* `LightMutex` (`InternalThread::_thread_lock`), not a
single global lock, so hot start/stop calls on different threads don't
contend with each other; only structural changes (creating a new
collector/thread) take the client-wide `ReMutex _lock`.

**Client/server wire protocol: a TCP control channel plus a UDP data
channel.** Connecting (`PStatClientImpl::client_connect()`) opens a TCP
socket and immediately sends a `T_hello`
[`PStatClientControlMessage`](PStatClientControlMessage.md) (hostname,
progname, major/minor protocol version); the server replies with its own
`T_hello` [`PStatServerControlMessage`](PStatServerControlMessage.md)
carrying the UDP port to send frame data to — until that reply arrives
(`_got_udp_port == false`), no thread is marked active and no frame data is
recorded. Per-frame timing/level data
([`PStatFrameData`](PStatFrameData.md)) is sent mostly over UDP, with a
sine/cosine-weighted ratio (`pstats-tcp-ratio`) of packets deliberately
routed over TCP instead, and any datagram too large for a single UDP packet
falls back to TCP automatically (further throttling that thread's send rate
proportionally to how many UDP-sized chunks it would have taken). Sends are
rate-limited to `pstats-max-rate` packets/sec/thread
(`PStatClientImpl::transmit_frame_data()`). New collectors and threads
created after the initial connect are pushed incrementally via
`T_define_collectors`/`T_define_threads` messages (batched at ≤700
collectors per message — empirically found to stay under the 64K datagram
limit); `T_expire_thread` exists in the message `enum` but nothing in this
module currently sends it. If the UDP socket ever reports a connection
reset, the client permanently falls back to TCP-only for the rest of that
session.

**Protocol version gating.** [`PStatClientVersion`](PStatClientVersion.md)'s
major version must match the server exactly; the minor version only needs to
be `<=` the server's. The current protocol is major 3, minor 0. Enabling
`pstats-python-profiler` forces the client to advertise at least 3.1 (added
specifically to support the nested start/stop pairs the Python profiler
integration generates); 3.2 (not currently forced by anything) added 32-bit
frame-data counts and `T_expire_thread`.

**A large static table assigns default display properties to well-known
collector names.** `pStatProperties.cxx` defines two parallel tables,
`time_properties[]` and `level_properties[]` — each a flat array of
`{active-by-default, full dotted/colon name, suggested RGB color, [units,
suggested scale, ...]}` entries for collectors the engine itself creates
(`"Frame"`, `"Cull"`, `"Draw:Set State"`, `"System memory:Heap"`, ...).
`initialize_collector_def()` looks up a `PStatCollectorDef` by its full
colon-separated name against both tables when the def is first materialized
(lazily, on first real use of a collector), then layers a matching set of
per-collector `Config.prc` variables on top (`pstats-active-<name>`,
`pstats-color-<name>`, `pstats-sort-<name>`, etc., with `:`/space converted
to `-`) so any collector's display can be overridden without a recompile.
Collectors not in either table just get engine defaults (`_sort = -1`, no
suggested color).

**Config variables** (`config_pstatclient.h`, notify category **`pstats`** —
not `pstatclient`, an easy mismatch to trip over): `pstats-name` (client
display name), `pstats-host`/`pstats-port` (default `5185` — moved off 5180
because that port's used by AIM), `pstats-max-rate`, `pstats-threaded-write`
+ `pstats-max-queue-size` (write to the PStats socket from a sub-thread, and
how many queued packets before the writer starts dropping them),
`pstats-tcp-ratio`, `pstats-target-frame-rate` (cosmetic marker line on the
server graph only), `pstats-gpu-timing`, `pstats-python-profiler`,
`pstats-mem-other` (fold `DO_MEMORY_USAGE` categories under 0.1% of the
total into an `"...:Other"` bucket instead of showing each individually).
Three more (`pstats-scroll-mode`, `pstats-history`, `pstats-average-time`)
are read by this module but only affect the *server's* display, not
anything client-side.

**`PStatClient::main_tick()` also drives `DO_MEMORY_USAGE` reporting when
that's compiled in.** Separately from the normal collector API, `main_tick()`
walks `TypeRegistry`'s per-`TypeHandle` memory-class accounting (singleton
heap allocations, array allocations, and the two "deleted chain"
active/inactive pools) each frame, creating one `"System memory:<category>:
<TypeName>"` level collector per type that exceeds the `pstats-mem-other`
threshold and bucketing everything smaller into a shared `"...:Other"`
collector per category.

## File index

| Class | Purpose |
|---|---|
| [PStatClient.md](PStatClient.md) | Client-side singleton: collector/thread registry, connect/tick entry points |
| [PStatClientImpl.md](PStatClientImpl.md) | Internal TCP/UDP protocol implementation behind `PStatClient` |
| [PStatCollector.md](PStatCollector.md) | Lightweight handle app code constructs to time/level-track a section of code |
| [PStatCollectorDef.md](PStatCollectorDef.md) | Name/color/units/sort metadata for one collector |
| [PStatCollectorForward.md](PStatCollectorForward.md) | Cheap forward-reference handle for pre-link-time use |
| [PStatThread.md](PStatThread.md) | Per-`Thread` handle with independent timing/level state |
| [PStatTimer.md](PStatTimer.md) | RAII start/stop wrapper around a `PStatCollector` |
| [PStatFrameData.md](PStatFrameData.md) | Raw per-frame start/stop/level event buffer |
| [PStatClientVersion.md](PStatClientVersion.md) | Protocol major/minor version negotiation |
| [PStatClientControlMessage.md](PStatClientControlMessage.md) | Client→server TCP control message |
| [PStatServerControlMessage.md](PStatServerControlMessage.md) | Server→client TCP control message |

## Status

pstatclient — done (2026-08-25). Other `panda/src/*` subsystems not yet
documented — see `../../README.md` for the overall index.
