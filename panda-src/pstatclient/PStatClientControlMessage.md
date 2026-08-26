# PStatClientControlMessage

**Source:** `panda/src/pstatclient/pStatClientControlMessage.{h,cxx}`
**Inherits from:** none

The client→server half of the TCP control protocol — sent whenever the
client needs to announce itself, or tell the server about new collectors or
threads created since the last announcement. Distinct from the per-frame
timing/level data (which goes through
[`PStatFrameData`](PStatFrameData.md), mostly over UDP) — this message type
only ever travels over the TCP control channel.

## Behavior

**One `Type` enum drives both `encode()`'s and `decode()`'s switch — each
case only touches the fields relevant to that message type**, since all
fields for every type live in the same flat struct-like class rather than a
tagged union. `T_hello` writes hostname/progname/major/minor version;
`T_define_collectors` writes a count followed by each
[`PStatCollectorDef`](PStatCollectorDef.md)'s own `write_datagram()` output;
`T_define_threads` writes a starting index plus a list of thread names;
`T_expire_thread` writes just a starting index. `T_datagram` (index 0) is
explicitly *not* a real control message type — it exists so the shared byte
that distinguishes a control message from a frame-data datagram
(`PStatClientImpl` prefixes control datagrams differently from frame data)
has a defined "not a control message" value.

**`decode()` tolerates an older, version-less `T_hello` payload.** If the
datagram runs out of data right after the hostname/progname strings
(`get_remaining_size() == 0`), it defaults to major/minor version `1.0`
instead of failing — a backward-compatibility path for clients built before
version numbers existed in the protocol at all (per
[`PStatClientVersion.md`](PStatClientVersion.md)'s history, that's pre-2.0,
from before 2001-05-18).

**`T_expire_thread` is defined in the enum and encoded/decoded, but nothing
else in this module currently constructs or sends one** — it was added in
protocol 3.2 alongside 32-bit data counts, but appears to be a hook for the
server side (or a future client-side feature) rather than something
`PStatClientImpl` emits today.

## API reference

```cpp
class PStatClientControlMessage {
public:
  PStatClientControlMessage();

  void encode(Datagram &datagram) const;
  bool decode(const Datagram &datagram, PStatClientVersion *version);

  enum Type {
    T_datagram = 0,
    T_hello,
    T_define_collectors,
    T_define_threads,
    T_expire_thread,
    T_invalid
  };

  Type _type;

  // Used for T_hello
  std::string _client_hostname;
  std::string _client_progname;
  int _major_version;
  int _minor_version;

  // Used for T_define_collectors
  pvector<PStatCollectorDef *> _collectors;

  // Used for T_define_threads
  int _first_thread_index;
  pvector<std::string> _names;
};
```

- `decode()`'s `PStatCollectorDef*` entries for `T_define_collectors` are
  heap-allocated by the decoder (`new PStatCollectorDef`) and left for the
  caller to own/free — there's no destructor cleanup of `_collectors` in
  this class itself.
- Default-constructs with `_type = T_invalid`, so a message that's never
  explicitly typed fails cleanly if encoded by mistake (logged via
  `pstats_cat.error()`).

## Usage

Constructed and sent internally by
[`PStatClientImpl`](PStatClientImpl.md)'s `send_hello()`,
`report_new_collectors()`, and `report_new_threads()` — not something
application code builds directly.

## Related classes

- [`PStatServerControlMessage`](PStatServerControlMessage.md) — the
  server→client counterpart of this handshake
- [`PStatCollectorDef`](PStatCollectorDef.md) — the payload of
  `T_define_collectors`
- [`PStatClientVersion`](PStatClientVersion.md) — passed into `decode()` to
  resolve version-dependent payload shapes
