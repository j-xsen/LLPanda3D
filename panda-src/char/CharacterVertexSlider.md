# CharacterVertexSlider

**Source:** `panda/src/char/characterVertexSlider.h` / `.I` / `.cxx`
**Inherits:** [../gobj/VertexSlider.md](../gobj/VertexSlider.md)

A specialization of `VertexSlider` that returns the current value of a
particular [CharacterSlider](CharacterSlider.md). This is the object a
`GeomVertexData`'s `SliderTable` actually references — the `SliderTable`
never points at a `CharacterSlider` directly.

## Behavior notes

- **`get_slider()` reads the underlying value directly**, not through
  `CharacterSlider::get_value()` — it returns `_char_slider->_value`
  (accessible because `CharacterVertexSlider` reaches into a friend-declared
  private member), bypassing any public accessor entirely.
- **Registers/unregisters itself with the `CharacterSlider`** on
  construction/destruction (`_char_slider->_vertex_sliders.insert(this)` /
  `.erase(this)`), which is how the slider knows to call `mark_modified()`
  on this object whenever its value changes — see
  [CharacterSlider](CharacterSlider.md)'s Behavior notes. Consequently a
  `CharacterVertexSlider` must be destroyed before its underlying
  `CharacterSlider` (the slider's destructor asserts its observer set is
  empty).
- **The slider's `InternalName` is copied from the `CharacterSlider`'s
  name** (`InternalName::make(char_slider->get_name())`) — a `SliderTable`
  looks this object up by the same name as the underlying `CharacterSlider`
  part.
- **The `.cxx` constructor's doc comment is stale/copy-pasted** from
  [JointVertexTransform](JointVertexTransform.md)'s constructor ("converts
  vertices from the indicated joint's coordinate space, into the other
  indicated joint's space") — this class does no coordinate conversion at
  all; it simply wraps and forwards one `CharacterSlider`'s scalar value.
- **The default constructor is `private`**, used only by the bam loader;
  `complete_pointers()` re-derives `_name` from the resolved `_char_slider`
  once that pointer is available.

## API

| Signature | Notes |
|---|---|
| `CharacterVertexSlider(CharacterSlider *char_slider)` | Registers itself with `char_slider` |
| `INLINE const CharacterSlider *get_char_slider() const` | |
| `virtual PN_stdfloat get_slider() const` | Returns `char_slider->_value` directly — see Behavior notes |

## Usage

```cpp
#include "character.h"
#include "characterJointBundle.h"
#include "characterSlider.h"
#include "characterVertexSlider.h"
#include "sliderTable.h"
#include "sparseArray.h"

PT(Character) actor = new Character("Actor");
CharacterJointBundle *bundle = actor->get_bundle(0);
CharacterSlider *smile = new CharacterSlider(bundle, "smile", 0.0f);

PT(CharacterVertexSlider) vslider = new CharacterVertexSlider(smile);

PT(SliderTable) table = new SliderTable;
SparseArray affected_rows;  // rows this morph target touches
table->add_slider(vslider, affected_rows);

PN_stdfloat current = vslider->get_slider();
```

## See also

[CharacterSlider](CharacterSlider.md), [Character](Character.md),
[../gobj/VertexSlider.md](../gobj/VertexSlider.md), [../gobj/SliderTable.md](../gobj/SliderTable.md),
[JointVertexTransform](JointVertexTransform.md) (the analogous matrix-valued object for skinning),
[README.md](README.md)
