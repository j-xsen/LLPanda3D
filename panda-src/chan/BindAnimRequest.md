# BindAnimRequest

**Source:** `panda/src/chan/bindAnimRequest.h` / `.I` / `.cxx`
**Inherits:** `ModelLoadRequest` (`panda/src/pgraph`)

An `AsyncTask` that performs an asynchronous load-and-bind: load an
animation file off the main thread, then bind it to a waiting
[AnimControl](AnimControl.md) placeholder once loaded. Issued internally by
`PartBundle::load_bind_anim()` — not normally constructed directly by
application code.

## Behavior notes

- **Checks for abandonment before doing any work.** `do_task()`'s first
  substantive check after loading is `_control->get_ref_count() == 1` —
  meaning this `BindAnimRequest` (via `_control`) is the *only* remaining
  reference to the `AnimControl`. If application code has already dropped
  every other reference to the control before the async load finished, the
  request calls `fail_anim()` and bails without even attempting the bind —
  no one is left to observe the result anyway.
- **Every failure path converges on `_control->fail_anim(part)`.** Model
  file fails to load, loaded model has no `AnimBundleNode` inside it, or
  `PartBundle::do_bind_anim()` itself rejects the match (wrong hierarchy,
  incompatible joints) — all three call `fail_anim()` and return `DS_done`
  identically; there's no distinction visible to the caller about *which*
  of the three failure modes occurred beyond what the `chan` notify category
  logs.
- **`set_anim_model()` is called even on a path that may still fail later**
  — the loaded model is attached to the control (for refcounting purposes,
  see [AnimControl](AnimControl.md#behavior-notes)) as soon as it loads
  successfully, before the bundle-matching step is attempted.
- **Always returns `DS_done` in one pass** — despite being an `AsyncTask`,
  this isn't a multi-step/resumable task; `do_task()` fully resolves the
  bind attempt (success or failure) in a single call.

## API

| Signature | Notes |
|---|---|
| `BindAnimRequest(const std::string &name, const Filename &filename, const LoaderOptions &options, Loader *loader, AnimControl *control, int hierarchy_match_flags, const PartSubset &subset)` | Constructs the request; `control` is the pending placeholder to resolve |
| `virtual DoneStatus do_task()` | Loads the file, finds its `AnimBundle`, and attempts the bind; always resolves `control` one way or another |

## Usage

```cpp
// Typically issued indirectly:
PT(PartBundle) part = ...; // obtained from a PartBundleNode, see PartBundle.md
LoaderOptions options;
PT(AnimControl) control = part->load_bind_anim(
    window->get_graphics_engine()->get_default_loader(),
    "panda-walk4.egg", 0, options, true);
// control is a pending placeholder immediately; the real bind happens
// asynchronously via a BindAnimRequest queued on the loader.
control->wait_pending();
if (control->has_anim()) {
  control->loop(true);
}
```

## See also

[AnimControl](AnimControl.md), [PartBundle](PartBundle.md),
[PartSubset](PartSubset.md), [AnimBundleNode](AnimBundleNode.md)
