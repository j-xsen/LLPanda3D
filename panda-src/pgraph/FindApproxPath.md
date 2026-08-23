# FindApproxPath

**Source:** `panda/src/pgraph/findApproxPath.h` (+ `.I`, `.cxx`)

Internal (not exported outside `pgraph`) compiled representation of a
[NodePath](NodePath.md) search pattern, as passed to `NodePath::find()`,
`find_all_matches()`, and friends — e.g. `"**/character/+GeomNode"`.
`add_string()` parses a slash-separated path string into a sequence of
`Component`s, each specifying what a graph level must match: an exact or
glob name, an exact or inexact (base-class-inclusive) type, a tag
key/value, a wildcard single level (`*`), a wildcard zero-or-more levels
(`**`), or one specific node by pointer. A trailing `;flags` clause (e.g.
`;+h-s`) toggles hidden/stashed/case-insensitive matching for the whole
path via `add_flags()`.

## Behavior notes

- `@@` prefixing a component (e.g. `"@@light"`) marks that component as
  matching only a *stashed* node at that level.
- `add_match_name_glob()` silently downgrades to an exact-name match
  (`CT_match_name`) whenever the given string has no actual glob special
  characters — an optimization so simple path components don't pay glob
  matching overhead.
- `**` cannot be combined with `@@` directly (`@@**` is rejected as
  ambiguous) — the error message directs you to use `@@*/**` or `**/@@*`
  instead.

## FindApproxLevelEntry (internal traversal helper, folded in)

**Source:** `panda/src/pgraph/findApproxLevelEntry.h` (+ `.I`, `.cxx`)

`FindApproxPath` is a static, parsed pattern; `FindApproxLevelEntry` is the
per-node, per-breadth-first-level search state used while actually walking
the graph against it — one entry per `(node, pattern position)` pair still
under consideration. It has no public API (application code never touches
it) so it doesn't warrant a standalone doc, but its algorithm is worth
noting: `NodePath::find()` runs a **breadth-first** search level by level
(via linked `FindApproxLevelEntry` chains built from a
[WorkingNodePath](WorkingNodePath.md) at each step) rather than depth-first,
specifically to make `**` (match zero-or-more levels) tractable — an entry
positioned at a `**` component re-considers itself at the *same* graph
level with the pattern position advanced (supporting a zero-level match)
before also fanning out to children. `consider_node()` is the per-entry
step: it either records a solution (pattern fully matched) into the result
`NodePathCollection`, or spawns child entries for the next level; hidden
nodes are skipped unless the pattern's `+h`/default allows returning
hidden, and stashed children are only visited when the pattern requests
stashed matching (`;+s` or an `@@`-flagged component).

## API

| Method | Notes |
|---|---|
| `add_string(path)` | Parses a full `"a/b/**/c;+h-s"`-style path string |
| `add_flags(flags)` | Parses a trailing `;+h-s-i`-style flag string |
| `add_component(str)` | Parses one slash-delimited segment |
| `add_match_name` / `_name_glob` / `_exact_type` / `_inexact_type` / `_tag` / `_tag_value` / `_one` / `_many` / `_pointer` | Programmatic component builders, one per match kind |
| `get_num_components() const` | Length of the compiled pattern |
| `matches_component(index, node) const` | Test one pattern component against a node |
| `is_component_match_many(index) const` | Whether component `index` is a `**` wildcard |
| `matches_stashed(index) const` | Whether component `index` requires a stashed match (`@@`) |
| `return_hidden()` / `return_stashed()` / `case_insensitive()` | Global flags parsed from `;...` |

## See also

[NodePath](NodePath.md), [NodePathCollection](NodePathCollection.md), [WorkingNodePath](WorkingNodePath.md)
