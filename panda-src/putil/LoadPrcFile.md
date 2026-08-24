# load_prc_file (free functions)

**Source:** `panda/src/putil/load_prc_file.h` / `.cxx`

Convenience free functions for loading a `.prc` config file (or in-memory
config text) at runtime, as an "explicit page" in dtool's config system
(`ConfigPageManager`). Lives in `putil` rather than `dtool` specifically so
it can use the virtual file system and model path, both of which are
unavailable at the `dtool` layer.

## Behavior notes

- **`load_prc_file()` resolves the filename against two search paths in
  order**: the `ConfigPageManager`'s own configured search path first, then
  (if still unresolved) the model path — a convenience so a `.prc` sitting
  alongside your models is found without extra setup.
- **Reads through the virtual file system** (`VirtualFileSystem::open_read_file`),
  so it transparently works for a `.prc` packed inside a mounted multifile,
  not just a real disk file.
- On failure to open or parse, both loader functions log via the `util`
  notify category and return `nullptr` rather than throwing; on a parse
  failure specifically, `load_prc_file()` also cleans up by calling
  `delete_explicit_page()` on the page it had provisionally created.
- **`load_prc_file_data()` sets `trust_level(1)`** on success — the source
  comment flags this as a "temp hack" — meaning prc text supplied directly
  as a string is treated as more trusted than a file loaded from disk by
  `load_prc_file()` (which leaves the default trust level).
- Every returned `ConfigPage*` must eventually be passed to
  `unload_prc_file()` if you want it removed later; there's no automatic
  lifetime management tied to the caller.
- `hash_prc_variables()` is only compiled `#ifdef HAVE_OPENSSL` — it hashes
  the current effective config-variable state (via
  `ConfigVariableManager::write_prc_variables()`) into a `HashVal`, useful
  for detecting whether the effective config differs between two runs/machines.

## API

| Signature | Notes |
|---|---|
| `ConfigPage *load_prc_file(const Filename&)` | Searches prc search path, then model path; via VFS |
| `ConfigPage *load_prc_file_data(const std::string &name, const std::string &data)` | In-memory prc text; `name` is just a display label; sets trust_level 1 |
| `bool unload_prc_file(ConfigPage*)` | Invalidates the pointer; returns false if unknown |
| `void hash_prc_variables(HashVal&)` | `HAVE_OPENSSL` only |

## Usage

```cpp
ConfigPage *page = load_prc_file("config/app.prc");
// ... later, if you want to remove it ...
unload_prc_file(page);
```

## See also

[README.md](README.md)
