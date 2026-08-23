# LLPanda3D — Panda3D C++ API reference for Claude

A git-tracked, incrementally-built reference to the Panda3D 1.10.16 C++ engine
source (`panda/src/`), written so a Claude session can understand a
subsystem **without re-reading the raw engine source**. Official Panda3D
docs are Python-focused; this covers the C++ API.

## Setup

Clone this repo — that's it, no Panda3D source needed:

```
git clone https://github.com/j-xsen/LLPanda3D.git
```

Then tell your Claude session to read `LLPanda3D/README.md` first (or point
a `CLAUDE.md` at it). The docs are self-contained; source paths are noted
for reference only.

**If you also have the Panda3D 1.10.16 source** (not just the standard
distributable), cloning `LLPanda3D/` as a sibling of `panda/` unlocks one extra
step: Claude can cross-check a doc against the real source when something
looks off (see step 3 below). Without the source, skip that step — the docs
still work, Claude just can't verify them against ground truth.

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
3. *(Only if you have the Panda3D source locally.)* Doc vs. source mismatch:
   trust the source (`panda/src/<module>/`); treat the doc as stale and
   consider updating it.
4. Module not listed: read the source directly if you have it; consider
   adding a `panda-src/<module>/` section (pgui format) if it's worth
   preserving.

## Status

| Module | `panda/src/` path | Status |
|---|---|---|
| pgui | `panda/src/pgui` | Done (2026-08-22) — see [panda-src/pgui/README.md](panda-src/pgui/README.md) |
| event | `panda/src/event` | Done (2026-08-22) — see [panda-src/event/README.md](panda-src/event/README.md) |
| text | `panda/src/text` | Done (2026-08-22) — see [panda-src/text/README.md](panda-src/text/README.md) |
| framework | `panda/src/framework` | Done (2026-08-22) — see [panda-src/framework/README.md](panda-src/framework/README.md) |
| display | `panda/src/display` | Done (2026-08-22) — see [panda-src/display/README.md](panda-src/display/README.md) |
| pgraph | `panda/src/pgraph` | Done (2026-08-23) — see [panda-src/pgraph/README.md](panda-src/pgraph/README.md) |
| gobj | `panda/src/gobj` | Done (2026-08-23) — see [panda-src/gobj/README.md](panda-src/gobj/README.md) |

Everything else under `panda/src/` is not yet documented.
