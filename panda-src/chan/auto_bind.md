# auto_bind

**Source:** `panda/src/chan/auto_bind.h` / `.cxx`

A free function (not a class) that walks a scene graph subtree, finds every
[AnimBundle](AnimBundle.md) (via `AnimBundleNode`) and every
[PartBundle](PartBundle.md) (via `PartBundleNode`), groups them by name, and
binds each matching name-group with `PartBundle::bind_anim()`. It's the
standard shortcut for "I loaded a character model and some separate
animation files, now hook them up" without writing the matching logic
yourself.

## Behavior notes

- **Matching is purely by bundle name, many-to-many within a name group.**
  `r_find_bundles()` buckets every found `AnimBundle`/`PartBundle` into a
  `pmap<name, set<...>>`; `bind_anims()` then does a full cross-product bind
  attempt between every part and every anim sharing a name — so two
  differently-loaded parts both named `"panda"` will each attempt a bind
  against every anim also named `"panda"`.
- **Name collisions in the output collection get suffixed, not dropped.** If
  a bind succeeds under a name already stored in `controls`, `auto_bind()`
  synthesizes `"name.1"`, `"name.2"`, etc. (via `format_string()`) rather
  than overwriting the earlier entry or failing — so a scene with multiple
  differently-parented parts sharing an anim name ends up with all bindings
  preserved under distinct keys.
- **`hierarchy_match_flags` (from `PartGroup`, e.g.
  `HMF_ok_wrong_root_name`) gates a second, looser pass.** The first pass
  only matches parts/anims whose names are exactly equal. If
  `HMF_ok_wrong_root_name` is set, every part and anim that had *no* exact
  name match is collected into "extra" sets and *all* attempted against each
  other regardless of name in a second pass — this is what lets an anim
  bundle named differently from its target part still bind, at the cost of
  potentially attempting many more (mostly failing) bind calls.
- **A failed `bind_anim()` attempt is silent by default** — nothing is added
  to `controls`, no exception thrown; enable the `chan` notify category at
  `info` or `debug` level to see what was attempted and why particular pairs
  didn't bind.
- **The name stored in `controls` prefers the AnimControl's own name over
  the AnimBundle's** — `bind_anims()` uses `(*abi)->get_name()` (the found
  `AnimBundle`'s name, confusingly reusing `abi`'s dereference which is
  actually the bundle) and falls back to the anim's name only if empty; in
  practice both are normally the same "basename" set when the bundle was
  loaded.

## API

| Signature | Notes |
|---|---|
| `void auto_bind(PandaNode *root_node, AnimControlCollection &controls, int hierarchy_match_flags = 0)` | Walks `root_node`'s subtree, fills `controls` with every successful bind, named by anim/bundle name (deduplicated with `.N` suffixes) |

## Usage

```cpp
NodePath actor_root = window->load_model(window->get_render(), "panda-model");
window->load_model(actor_root, "panda-walk4");
window->load_model(actor_root, "panda-run");

AnimControlCollection anims;
auto_bind(actor_root.node(), anims);

anims.loop("panda-walk4", true);
```

## See also

[PartBundle](PartBundle.md), [AnimBundle](AnimBundle.md),
[AnimBundleNode](AnimBundleNode.md), [PartBundleNode](PartBundleNode.md),
[AnimControlCollection](AnimControlCollection.md), [PartGroup](PartGroup.md)
(hierarchy match flags)
