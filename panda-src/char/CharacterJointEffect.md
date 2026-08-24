# CharacterJointEffect

**Source:** `panda/src/char/characterJointEffect.h` / `.I` / `.cxx`
**Inherits:** [../pgraph/RenderEffect.md](../pgraph/RenderEffect.md)

An effect automatically attached to a node by
[CharacterJoint::add_net_transform()](CharacterJoint.md)/`add_local_transform()`
(and internally whenever a joint's owning character changes). It binds the
node back to its [Character](Character.md), so that querying the node's
relative transform will implicitly call `Character::update()` first — a
lazy update-on-query mechanism that ensures a node "socketed" to a joint
always reflects the current pose, even if read before the next cull
traversal's automatic update would otherwise have run.

## Behavior notes

- **Never constructed directly** — the constructor is `private`; use the
  static `make()` factory, which is itself only called from
  `CharacterJoint::add_net_transform()`/`add_local_transform()`/
  `set_character()`.
- **Holds only a weak pointer** (`WPT(Character) _character`) — attaching
  this effect does not keep the `Character` alive. `get_character()` calls
  `.lock()` and returns `nullptr` once the `Character` has been destroyed.
- **`matches_character()` exists because `get_character()` isn't always
  safe to call.** It compares `_character.get_orig() == character &&
  !_character.was_deleted()` without constructing a strong `PointerTo` —
  this can be called even while the `Character` is mid-destruction (ref
  count already `0` but not yet freed), which is exactly when
  `CharacterJoint::remove_net_transform()`/`remove_local_transform()` need
  to check whether an attached effect still belongs to this joint's
  character before clearing it.
- **`make()` works around a pointer-reuse interning hazard.** Effects are
  interned/uniqued by `compare_to_impl()`, which sorts purely by the raw
  `Character` pointer value. If a *new* `Character` object happens to be
  allocated at the exact address of a previously-deleted one that still has
  a lingering (now weakly-invalid) interned `CharacterJointEffect`,
  `return_new()` would hand back that stale instance pointing at the wrong,
  dead `Character`. `make()` detects this case
  (`!new_effect->_character.is_valid_pointer()`) and force-repairs the
  returned effect's weak pointer in place to point at the live `character`
  argument instead — documented in the `.cxx` as "a little weird."
- **`safe_to_transform()` returns `true`**, on the assumption that anything
  transforming the socketed joint node will also transform the `Character`
  node above it. **`safe_to_combine()` returns `false`** — each instance is
  specific to one node's binding and must not be merged with a sibling's
  during flattening.
- **`has_cull_callback()`/`cull_callback()` and
  `has_adjust_transform()`/`adjust_transform()` are both implemented, and
  both do the same thing:** call `character->update()` via the weak
  pointer (a no-op if it has expired), then substitute the node's own
  already-known transform back in. Neither hook actually modifies the
  transform — they exist purely to force the update side effect at the two
  points (cull time, and transform-flattening time) where the node's
  transform might otherwise be read stale.
- **`compare_to_impl()` intentionally ignores weak-pointer validity** and
  sorts only by `_character.get_orig()` — the `.cxx` notes that including
  validity in the sort would be unsafe, since it can change silently
  (when the `Character` is destroyed) and would invalidate this object's
  position in any map/set that used the comparison to order it.

## API

| Signature | Notes |
|---|---|
| `static CPT(RenderEffect) make(Character *character)` | The only way to construct one; see Behavior notes for the pointer-reuse repair |
| `INLINE PT(Character) get_character() const` | `.lock()`'s the weak pointer; `nullptr` if the character is gone |
| `INLINE bool matches_character(Character *character) const` | Safe identity check, usable even mid-destruction |
| `virtual bool safe_to_transform() const` | `true` |
| `virtual bool safe_to_combine() const` | `false` |
| `virtual bool has_cull_callback() const` / `cull_callback(...)` | Both `true`/implemented; forces `character->update()` |
| `virtual bool has_adjust_transform() const` / `adjust_transform(...)` | Both `true`/implemented; forces `character->update()` |

## Usage

```cpp
#include "character.h"
#include "characterJointBundle.h"
#include "characterJoint.h"
#include "characterJointEffect.h"
#include "pandaNode.h"

PT(Character) actor = new Character("Actor");
CharacterJointBundle *bundle = actor->get_bundle(0);
CharacterJoint *hand = actor->find_joint("hand");

if (hand != nullptr) {
  PT(PandaNode) prop_holder = new PandaNode("prop-holder");

  // Attaches a CharacterJointEffect(actor) to prop_holder automatically.
  hand->add_net_transform(prop_holder);

  CPT(RenderEffect) effect =
    prop_holder->get_effect(CharacterJointEffect::get_class_type());
  if (effect != nullptr) {
    const CharacterJointEffect *cje = DCAST(CharacterJointEffect, effect);
    PT(Character) owner = cje->get_character();
  }
}
```

## See also

[CharacterJoint](CharacterJoint.md), [Character](Character.md),
[../pgraph/RenderEffect.md](../pgraph/RenderEffect.md), [README.md](README.md)
