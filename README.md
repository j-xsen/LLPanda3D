# llpanda — Panda3D C++ API reference for Claude

A git-tracked, incrementally-built reference to the Panda3D 1.10.16 C++ engine
source (`../panda/src/`), written so a Claude session can understand a given
subsystem's classes, methods, and usage **without re-reading the raw engine
source**. Official Panda3D docs are Python-focused; this covers the C++ API
surface specifically.

## How this is organized

`panda-src/<module>/` mirrors a directory under `panda/src/` (e.g.
`panda-src/pgui/` documents `panda/src/pgui/`). Each module has its own
`README.md` overview (class hierarchy, cross-cutting concepts) plus one
`.md` file per class, cross-linked with relative markdown links. See
[`panda-src/pgui/README.md`](panda-src/pgui/README.md) for the reference
example of this format.

Each class doc states its source path (`.h`/`.I`/`.cxx`), inheritance,
purpose, non-obvious behavior notes pulled from the `.cxx` (not just the
header), a compact API reference, events it throws (if any), a minimal usage
snippet, and links to related classes.

## How to use this as a Claude session

1. Check the status table below for which modules are documented.
2. Read that module's `README.md` first for the shared vocabulary/concepts,
   then the specific class doc(s) you need.
3. If something looks like it may have changed since a doc was written (a
   named method/field doesn't match what you find in the real source), trust
   the source — `panda/src/<module>/` — and treat the doc as stale; consider
   updating it.
4. If a module isn't listed yet, it hasn't been documented — read the source
   directly for that one, and consider adding a `panda-src/<module>/` section
   following the pgui format if the work is substantial enough to be worth
   preserving.

## Status

| Module | `panda/src/` path | Status |
|---|---|---|
| pgui | `panda/src/pgui` | Done (2026-08-22) — see [panda-src/pgui/README.md](panda-src/pgui/README.md) |
| event | `panda/src/event` | Done (2026-08-22) — see [panda-src/event/README.md](panda-src/event/README.md) |

Everything else under `panda/src/` is not yet documented.
