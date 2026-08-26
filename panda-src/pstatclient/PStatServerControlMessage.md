# PStatServerControlMessage

**Source:** `panda/src/pstatclient/pStatServerControlMessage.{h,cxx}`
**Inherits from:** none

The server→client half of the TCP control protocol. Much smaller than
[`PStatClientControlMessage`](PStatClientControlMessage.md) — the server has
only one thing it needs to tell the client during the handshake: its own
identity and which UDP port to send frame data to.

## Behavior

**Only one real message type exists (`T_hello`); everything else is treated
as an error.** Unlike the client-side message, there's no
version/collector/thread negotiation flowing this direction — the server's
`T_hello` reply carries its hostname, program name, and the UDP port for the
client to start sending frame data to. `PStatClientImpl::
handle_server_control_message()` uses that reply to set `_got_udp_port =
true` — until this message arrives, the client holds every thread inactive
and records no frame data at all (see the module
[README](README.md)'s wire-protocol section).

**No version negotiation on this side of the handshake.** `decode()` takes
only a `Datagram`, no `PStatClientVersion*` — unlike
`PStatClientControlMessage::decode()`, this message's format has apparently
never needed a version-dependent field, so there's nothing to branch on.

## API reference

```cpp
class PStatServerControlMessage {
public:
  PStatServerControlMessage();

  void encode(Datagram &datagram) const;
  bool decode(const Datagram &datagram);

  enum Type {
    T_invalid,
    T_hello,
  };

  Type _type;

  // Used for T_hello
  std::string _server_hostname;
  std::string _server_progname;
  int _udp_port;
};
```

- Default-constructs with `_type = T_invalid`, matching
  `PStatClientControlMessage`'s convention.
- `decode()` returns `false` and logs an error for any type other than
  `T_hello` — there is currently no other valid server-to-client control
  message.

## Usage

Decoded internally by `PStatClientImpl::transmit_control_data()`'s read loop
and dispatched to `handle_server_control_message()` — application code never
constructs or handles this type directly. Encoding (`encode()`) exists for
symmetry / potential server-side reuse of this shared class, but nothing on
the client side of this module calls it.

## Related classes

- [`PStatClientControlMessage`](PStatClientControlMessage.md) — the
  client→server counterpart of this handshake
- [`PStatClientImpl`](PStatClientImpl.md) — decodes and reacts to this
  message
