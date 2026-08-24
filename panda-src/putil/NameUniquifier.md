# NameUniquifier

**Source:** `panda/src/putil/nameUniquifier.h` / `.I` / `.cxx`
**Inherits:** none
**Inherited by:** none

Accumulates a set of names and, for each one submitted via `add_name()`,
returns a name guaranteed unique among everything submitted so far —
generating a numbered variant if the requested name is empty or a repeat.
Used by exporters/converters that write to formats requiring unique names
(e.g. egg vertex pool names, node names in a format with no duplicate-name
tolerance).

## Behavior notes

- **Not a rename-in-place utility — it's a name generator with memory.**
  Construct one `NameUniquifier`, feed it every candidate name across the
  whole export/conversion pass (one instance, not one per name), and use
  its return value as the actual name to write out.
- **First-come-first-served: the *first* occurrence of a name is returned
  unchanged**, tracked in an internal `pset<string>`. Only the second and
  later occurrences of the same string get renamed.
- **Renaming scheme:** `prefix + separator + N` where `N` is a
  monotonically increasing counter (shared across *all* generated names in
  this instance, not per-prefix) that keeps incrementing until the
  resulting string is itself not already taken — so a generated name can in
  rare cases still collide with a not-yet-seen original name and will skip
  forward until it finds a free one.
- **Two-argument `add_name(name, prefix)` decouples "what gets checked
  against the uniqueness set" from "what to base a generated replacement
  name on"** — pass a different `prefix` than `name` when you want, e.g.,
  the generated fallback to look like a category name (`"joint-3"`) rather
  than echo the disambiguated string. If `name` is unique on first try, it
  is still returned as-is regardless of `prefix`; `prefix` only matters on
  the fallback path.
- **Empty-name handling:** if `prefix` is empty, the generated name is
  `empty_string + N` (no separator); if `empty` was never given a value at
  construction, it defaults to the `separator` string, so an empty name
  becomes e.g. `"_1"` if separator was `"_"`.
- **Uniqueness checking uses `set::insert()`'s return value directly**
  (`_names.insert(name).second`) rather than a separate `find()` +
  `insert()`, so it's a single hash lookup per attempt, not two.

## API

| Signature | Notes |
|---|---|
| `NameUniquifier(const std::string &separator = "", const std::string &empty = "")` | `empty` defaults to `separator` if left empty |
| `std::string add_name(const std::string &name)` | Equivalent to `add_name(name, name)` |
| `std::string add_name(const std::string &name, const std::string &prefix)` | `prefix` used only when a fallback name must be generated |

## Usage

```cpp
NameUniquifier uniq("_");     // separator "_", empty defaults to "_" too

uniq.add_name("joint");       // -> "joint"     (first occurrence, unchanged)
uniq.add_name("joint");       // -> "joint_1"   (duplicate, generated)
uniq.add_name("");            // -> "_2"        (empty name, generated)
uniq.add_name("hip", "bone"); // -> "hip"       (unique, prefix unused)
uniq.add_name("hip", "bone"); // -> "bone_3"    (duplicate; falls back to prefix)
```

## See also

[UniqueIdAllocator.md](UniqueIdAllocator.md) (a different "make something
unique" problem — recycled integer IDs, not names) · [README.md](README.md)
