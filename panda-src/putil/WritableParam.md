# WritableParam

**Source:** `panda/src/putil/writableParam.h` / `.I` / `.cxx`
**Inherits:** [FactoryParam](Factory.md#factoryparam)

A minimal [FactoryParam](Factory.md#factoryparam) that carries nothing but a
`const Datagram &` — the raw bytes a `TypedWritable` needs to construct
itself. It's the generic building block the `Factory<TypedWritable>`
mechanism was designed around; in practice, code going through
[BamReader](BamReader.md) uses the richer [BamReaderParam](BamReader.md)
instead (which additionally carries a `BamReader*` for pointer/PTA
resolution), so `WritableParam` mainly matters if you're driving a
`Factory<TypedWritable>` directly, without the Bam pointer-resolution
machinery.

## Behavior notes

- **Holds the `Datagram` by reference, not by value** — the referenced
  `Datagram` must outlive the `WritableParam`. The default assignment
  operator is explicitly deleted (a reference member can't be reseated);
  copy-construction is fine and just rebinds to the same datagram.

## API

| Signature | Notes |
|---|---|
| `WritableParam(const Datagram &datagram)` | |
| `const Datagram &get_datagram()` | |

## See also

[Factory.md](Factory.md) (FactoryParam / FactoryParams) ·
[BamReader.md](BamReader.md#api) (`BamReaderParam` — the variant actually
used by the Bam system) · [README.md](README.md)
