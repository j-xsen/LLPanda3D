# PStatCollectorDef

**Source:** `panda/src/pstatclient/pStatCollectorDef.{h,cxx}`
**Inherits from:** none

Plain-data metadata record for a single collector: its name, parent index,
suggested display color, sort order, level units/scale, and whether it's
active by default. One `PStatCollectorDef` backs each
[`PStatCollector`](PStatCollector.md) index, created lazily
(`PStatClient::Collector::make_def()`) the first time anything actually
needs it — not at collector-creation time.

## Behavior

**Defaults are plain zero/identity values until `set_parent()` and/or the
predefined-properties table fill them in.** The default constructor sets
`_sort = -1` (unsorted), `_factor = 1.0` (no unit conversion), `_is_active =
true`, and black (`0,0,0`) as the suggested color. `set_parent()` (called
right after construction, before `initialize_collector_def()`) inherits
`_level_units`, `_suggested_scale`, `_factor`, `_is_active`, and
`_active_explicitly_set` from the parent def — a child collector starts out
looking like its parent until the properties table or a `pstats-*` config
var overrides it.

**`initialize_collector_def()` (in `pStatProperties.cxx`, not a member of
this class) is what actually populates a meaningful color/sort/units.** It
looks the collector's full colon-separated name up in the compiled-in
`time_properties[]`/`level_properties[]` tables first, then layers
per-collector `Config.prc` variables (`pstats-active-<name>`,
`pstats-color-<name>`, `pstats-sort-<name>`, `pstats-scale-<name>`,
`pstats-units-<name>`, `pstats-factor-<name>`, with the name lowercased and
`:`/whitespace turned into `-`) on top — see the module
[README](README.md)'s "Core concepts" for the full table-lookup mechanism.
`_active_explicitly_set` tracks whether *any* config source has explicitly
set the active flag, so a later inherited/table default doesn't silently
override an explicit `pstats-active-foo false`.

**`write_datagram()`/`read_datagram()` are this class's wire format**, used
by [`PStatClientControlMessage`](PStatClientControlMessage.md)'s
`T_define_collectors` payload to describe new collectors to the server: index
and parent index as `int16`, name and level-units as length-prefixed
strings, color as three `float32`s, sort as `int16`, scale and factor as
`float32`. `read_datagram()` ignores the `PStatClientVersion` argument
entirely (unlike [`PStatFrameData::read_datagram()`](PStatFrameData.md),
whose wire format did change across protocol versions).

## API reference

```cpp
class PStatCollectorDef {
public:
  PStatCollectorDef();
  PStatCollectorDef(int index, const std::string &name);
  void set_parent(const PStatCollectorDef &parent);

  void write_datagram(Datagram &destination) const;
  void read_datagram(DatagramIterator &source, PStatClientVersion *version);

  struct ColorDef {
    float r, g, b;
  };

  int _index;
  std::string _name;
  int _parent_index;
  ColorDef _suggested_color;
  int _sort;
  std::string _level_units;
  double _suggested_scale;
  double _factor;
  bool _is_active;
  bool _active_explicitly_set;
};
```

- All fields are public and mutated directly (by `PStatClient`,
  `initialize_collector_def()`, and the datagram methods) — there is no
  private state or accessor layer.
- `_factor` is a level-value scale applied by `PStatClient::set_level()`/
  `add_level()`/`get_level()` (multiply on the way in, divide on the way
  out) — set from `pstats-factor-<name>` or a table entry's `inv_factor`.

## Usage

Never constructed directly by application code — created and populated
internally by [`PStatClient`](PStatClient.md) the first time a collector's
definition is actually needed (display, or sending `T_define_collectors` to
the server).

## Related classes

- [`PStatCollector`](PStatCollector.md) — the handle whose index maps to one
  of these defs
- [`PStatClient`](PStatClient.md) — creates and owns all `PStatCollectorDef`s
  via its internal `Collector` struct
- [`PStatClientControlMessage`](PStatClientControlMessage.md) — carries
  these over the wire in `T_define_collectors`
