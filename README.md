# LLPanda3D — Panda3D C++ API reference for LLMs

A git-tracked, incrementally-built reference to the Panda3D 1.10.16 C++ engine
source (`panda/src/`), written so an LLM coding agent can understand a
subsystem **without re-reading the raw engine source**. Official Panda3D
docs are Python-focused; this covers the C++ API.

## Setup

Clone this repo — that's it, no Panda3D source needed:

```
git clone https://github.com/j-xsen/LLPanda3D.git
```

Then point your agent at `LLPanda3D/README.md` first (e.g. via a
`CLAUDE.md`/`AGENTS.md` or equivalent project-instructions file). The docs
are self-contained; source paths are noted for reference only.

**If you also have the Panda3D 1.10.16 source** (not just the standard
distributable), cloning `LLPanda3D/` as a sibling of `panda/` unlocks one extra
step: the agent can cross-check a doc against the real source when something
looks off (see step 3 below). Without the source, skip that step — the docs
still work, the agent just can't verify them against ground truth.

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

## How to use this as an LLM agent

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
| pgraphnodes | `panda/src/pgraphnodes` | Done (2026-08-23) — see [panda-src/pgraphnodes/README.md](panda-src/pgraphnodes/README.md) |
| audio | `panda/src/audio` | Done (2026-08-23) — see [panda-src/audio/README.md](panda-src/audio/README.md) |
| collide | `panda/src/collide` | Done (2026-08-23) — see [panda-src/collide/README.md](panda-src/collide/README.md) |
| chan | `panda/src/chan` | Done (2026-08-23) — see [panda-src/chan/README.md](panda-src/chan/README.md) |
| char | `panda/src/char` | Done (2026-08-23) — see [panda-src/char/README.md](panda-src/char/README.md) |
| putil | `panda/src/putil` | Done (2026-08-23) — see [panda-src/putil/README.md](panda-src/putil/README.md) |
| linmath | `panda/src/linmath` | Done (2026-08-24) — see [panda-src/linmath/README.md](panda-src/linmath/README.md) |
| mathutil | `panda/src/mathutil` | Done (2026-08-24) — see [panda-src/mathutil/README.md](panda-src/mathutil/README.md) |
| dgraph | `panda/src/dgraph` | Done (2026-08-24) — see [panda-src/dgraph/README.md](panda-src/dgraph/README.md) |
| tform | `panda/src/tform` | Done (2026-08-24) — see [panda-src/tform/README.md](panda-src/tform/README.md) |
| cull | `panda/src/cull` | Done (2026-08-25) — see [panda-src/cull/README.md](panda-src/cull/README.md) |
| pipeline | `panda/src/pipeline` | Done (2026-08-25) — see [panda-src/pipeline/README.md](panda-src/pipeline/README.md) |
| pstatclient | `panda/src/pstatclient` | Done (2026-08-25) — see [panda-src/pstatclient/README.md](panda-src/pstatclient/README.md) |
| express | `panda/src/express` | Not started |
| downloader | `panda/src/downloader` | Not started |
| net | `panda/src/net` | Not started |
| nativenet | `panda/src/nativenet` | Not started |
| device | `panda/src/device` | Not started |
| recorder | `panda/src/recorder` | Not started |
| egg | `panda/src/egg` | Not started |
| egg2pg | `panda/src/egg2pg` | Not started |
| parametrics | `panda/src/parametrics` | Not started |
| particlesystem | `panda/src/particlesystem` | Not started |
| physics | `panda/src/physics` | Not started |
| grutil | `panda/src/grutil` | Not started |
| distort | `panda/src/distort` | Not started |
| movies | `panda/src/movies` | Not started |
| vision | `panda/src/vision` | Not started |
| ffmpeg | `panda/src/ffmpeg` | Not started |
| audiotraits | `panda/src/audiotraits` | Not started |
| vrpn | `panda/src/vrpn` | Not started |
| bullet | `panda/src/bullet` | Not started |
| ode | `panda/src/ode` | Not started |
| physx | `panda/src/physx` | Not started |
| speedtree | `panda/src/speedtree` | Not started |
| rocket | `panda/src/rocket` | Not started |
| pnmimage | `panda/src/pnmimage` | Not started |
| pnmimagetypes | `panda/src/pnmimagetypes` | Not started |
| pnmtext | `panda/src/pnmtext` | Not started |
| gsgbase | `panda/src/gsgbase` | Not started |
| glstuff | `panda/src/glstuff` | Not started |
| glgsg | `panda/src/glgsg` | Not started |
| gles2gsg | `panda/src/gles2gsg` | Not started |
| glesgsg | `panda/src/glesgsg` | Not started |
| dxgsg9 | `panda/src/dxgsg9` | Not started |
| tinydisplay | `panda/src/tinydisplay` | Not started |
| windisplay | `panda/src/windisplay` | Not started |
| wgldisplay | `panda/src/wgldisplay` | Not started |
| x11display | `panda/src/x11display` | Not started |
| glxdisplay | `panda/src/glxdisplay` | Not started |
| egldisplay | `panda/src/egldisplay` | Not started |
| cocoadisplay | `panda/src/cocoadisplay` | Not started |
| osxdisplay | `panda/src/osxdisplay` | Not started |
| iphone | `panda/src/iphone` | Not started |
| iphonedisplay | `panda/src/iphonedisplay` | Not started |
| android | `panda/src/android` | Not started |
| androiddisplay | `panda/src/androiddisplay` | Not started |
| dxml | `panda/src/dxml` | Not started |
| configfiles | `panda/src/configfiles` | Not started |
| downloadertools | `panda/src/downloadertools` | Not started |
| pandabase | `panda/src/pandabase` | Not started |
| skel | `panda/src/skel` | Not started |
| testbed | `panda/src/testbed` | Not started |
| doc | `panda/src/doc` | Not started (Panda3D's own doc source, likely nothing to mirror) |

Everything above is tracked; anything under `panda/src/` missing from this table hasn't been triaged yet.
