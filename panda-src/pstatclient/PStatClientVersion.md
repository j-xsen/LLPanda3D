# PStatClientVersion

**Source:** `panda/src/pstatclient/pStatClientVersion.{h,I,cxx}`
**Inherits from:** `ReferenceCount`

Records the protocol major/minor version a particular remote client (or, on
the client side, the connected server) is running, so version-dependent wire
format decisions (like [`PStatFrameData`](PStatFrameData.md)'s event-count
field width) can be made correctly.

## Behavior

**Default-constructs to "the version this build of Panda speaks."** The
constructor calls the two free functions in `pStatProperties.cxx`,
`get_current_pstat_major_version()`/`get_current_pstat_minor_version()`,
which return compile-time constants (currently major 3, minor 0) tracked in
a comment history in that file: 2.0 (version numbers first added), 2.1 (TCP
frame data support), 3.0 (32-bit TCP headers), 3.1 (nested start/stop pair
support, for the Python profiler integration — see
[`PStatClient.md`](PStatClient.md)), 3.2 (32-bit data counts,
`T_expire_thread` message type).

**`is_at_least()` treats major and minor version independently, not as a
combined ordinal.** It returns true if the major version is strictly
greater, *or* the major version matches exactly and the minor version is
`>=` the requested one — so a version 4.0 is "at least" 3.9, but a version
3.1 is not "at least" 2.99 by any numeric-combination trick; the comparison
is genuinely two-part. This is the check
[`PStatFrameData::read_datagram()`](PStatFrameData.md) uses to decide
between 16-bit and 32-bit event counts (`is_at_least(3, 2)`), and the one
`PStatClientImpl::send_hello()` uses to force at least 3.1 when
`pstats-python-profiler` is enabled.

## API reference

```cpp
class PStatClientVersion : public ReferenceCount {
public:
  PStatClientVersion();

  int get_major_version() const;
  int get_minor_version() const;

  void set_version(int major_version, int minor_version);

  bool is_at_least(int major_version, int minor_version) const;
};
```

- `set_version()` is used after decoding a peer's advertised version out of
  a `T_hello` message, to override the constructor's "this build's own
  version" default with whatever the peer actually reported.
- Reference-counted (`PT(PStatClientVersion)`) since it's typically handed
  around by pointer alongside a connection's lifetime rather than copied.

## Usage

Not constructed by typical application code — created internally when
processing a `T_hello` handshake, and passed down into
[`PStatFrameData::read_datagram()`](PStatFrameData.md) /
[`PStatCollectorDef::read_datagram()`](PStatCollectorDef.md) so those
methods can adapt their wire format to whichever protocol version the peer
is speaking.

## Related classes

- [`PStatFrameData`](PStatFrameData.md) — its `read_datagram()` branches on
  `is_at_least(3, 2)`
- [`PStatClientControlMessage`](PStatClientControlMessage.md) — carries the
  major/minor version numbers in its `T_hello` payload
- [`PStatClientImpl`](PStatClientImpl.md) — negotiates the version during
  `send_hello()`
