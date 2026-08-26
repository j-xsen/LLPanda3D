# PStatCollectorForward

**Source:** `panda/src/pstatclient/pStatCollectorForward.{h,I,cxx}`
**Inherits from:** `PStatCollectorForwardBase` (from `express`, undocumented here)

A cheap wrapper that lets a class defined *before* `pstatclient` in the link
order still report level data to a `PStatCollector`, by holding it behind
`PStatCollectorForwardBase`'s virtual interface instead of a direct
dependency. Exists purely to break a dependency-direction problem: something
low-level (in `express` or below) that wants to report stats can accept a
`PStatCollectorForwardBase*` without needing to link against
`pstatclient` itself; only the *caller*, which already depends on
`pstatclient`, constructs the concrete `PStatCollectorForward` and hands it
down.

## Behavior

**Wraps a `PStatCollector` by value and forwards exactly one operation.**
The constructor stores the given `PStatCollector` in `_col` (only present
when `DO_PSTATS` is defined — the field itself is `#ifdef`'d out otherwise,
matching the pattern of every other class in this module); the sole override,
`add_level(double level)`, calls `_col.add_level_now(level)` — the
*immediate*-flush variant, not the batching `add_level()`, since there's no
guarantee the low-level caller will ever call anything that would trigger an
implicit flush.

## API reference

```cpp
class PStatCollectorForward : public PStatCollectorForwardBase {
PUBLISHED:
  PStatCollectorForward(const PStatCollector &col);

  virtual void add_level(double level);
};
```

- `add_level()` is the only method — everything else this class needs
  (construction, storage) is private/internal.

## Usage

Constructed by code that already depends on `pstatclient` and needs to hand
a stats-reporting hook down to code that doesn't:

```cpp
static PStatCollector mem_pcollector("System memory:SomeSubsystem");
some_low_level_object->set_pstats_forward(
    new PStatCollectorForward(mem_pcollector));
// ...low-level code, without any pstatclient dependency, can now call
// forward->add_level(delta) through the PStatCollectorForwardBase interface.
```

## Related classes

- [`PStatCollector`](PStatCollector.md) — the real collector this wraps
- `PStatCollectorForwardBase` (in `express`) — the abstract interface that
  makes this indirection possible
