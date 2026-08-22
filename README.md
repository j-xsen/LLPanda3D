# llpanda — Panda3D C++ API reference for Claude

A git-tracked, incrementally-built reference to the Panda3D 1.10.16 C++ engine
source (`../panda/src/`), written so a Claude session can understand a
subsystem **without re-reading the raw engine source**. Official Panda3D
docs are Python-focused; this covers the C++ API.

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

1. Check the status table below for documented modules.
2. Read the module's `README.md` first, then the specific class doc(s).
3. Doc vs. source mismatch: trust the source (`panda/src/<module>/`); treat
   the doc as stale and consider updating it.
4. Module not listed: read the source directly; consider adding a
   `panda-src/<module>/` section (pgui format) if it's worth preserving.

## Status

| Module | `panda/src/` path | Status |
|---|---|---|
| pgui | `panda/src/pgui` | Done (2026-08-22) — see [panda-src/pgui/README.md](panda-src/pgui/README.md) |
| event | `panda/src/event` | Done (2026-08-22) — see [panda-src/event/README.md](panda-src/event/README.md) |
| text | `panda/src/text` | Done (2026-08-22) — see [panda-src/text/README.md](panda-src/text/README.md) |
| framework | `panda/src/framework` | Done (2026-08-22) — see [panda-src/framework/README.md](panda-src/framework/README.md) |
| display | `panda/src/display` | Done (2026-08-22) — see [panda-src/display/README.md](panda-src/display/README.md) |

Everything else under `panda/src/` is not yet documented.
