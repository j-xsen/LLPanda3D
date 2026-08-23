# VertexSlider

**Source:** `panda/src/gobj/vertexSlider.h` (+ `.I`, `.cxx`)
**Inherits:** `TypedWritableReferenceCount` **Inherited by:** `UserVertexSlider` (see below)

Abstract base for "produces a single float, recomputed on demand" nodes,
used to drive morph target (blend shape) animation — the same role
[VertexTransform](VertexTransform.md) plays for skeleton skinning, but
holding one scalar weight (typically 0.0–1.0) instead of a full 4x4 matrix.
Referenced by name from a [SliderTable](SliderTable.md), which a
`GeomVertexData` set up for CPU morph animation consults to know how far to
blend each morph delta column in.

## Behavior notes

- Pure virtual `get_slider() const = 0`; every subclass must supply the
  current value.
- Carries an `InternalName` (`get_name()`) set at construction and never
  changed — this is the name a `SliderTable` looks entries up by (unlike
  `VertexTransform`/`TransformTable`, which are index-based).
- Same global-modification-sequence mechanism as `VertexTransform`:
  a protected `mark_modified(Thread*)` bumps this slider's own sequence
  number and notifies every registered [SliderTable](SliderTable.md)
  holding a back-pointer to it (`_tables`, a `pset`, populated when a
  table containing this slider is registered).

## API

| Method | Notes |
|---|---|
| `explicit VertexSlider(const InternalName *name)` | Name is fixed for the object's lifetime. |
| `get_name()` | The `InternalName` this slider is looked up by. |
| `virtual get_slider() const = 0` | The current scalar value — override in subclasses. |
| `get_modified(Thread*)` | This slider's last-changed sequence number. |
| `mark_modified(Thread*)` *(protected)* | Subclasses call this after the returned value changes. |
| `output(ostream&)` / `write(ostream&, indent_level)` | Debug dump. |

## `UserVertexSlider` (folded subclass)

**Source:** `panda/src/gobj/userVertexSlider.h` (+ `.I`, `.cxx`)

A trivial `VertexSlider` subclass holding an app-supplied constant float,
set via `set_slider()`. Like `UserVertexTransform`, "rarely used except for
testing" — real morph-target animation typically drives sliders from
character animation data instead. `set_slider()` calls `mark_modified()`
internally.

| Method | Notes |
|---|---|
| `UserVertexSlider(const std::string &name)` / `UserVertexSlider(const InternalName *name)` | |
| `set_slider(PN_stdfloat)` | Replace the constant value; triggers `mark_modified()`. |
| `get_slider() const` (override) | Returns the stored value. |

## Usage

```cpp
PT(UserVertexSlider) smile = new UserVertexSlider("smile");
smile->set_slider(0.5f);
PT(SliderTable) table = new SliderTable;
table->add_slider(smile, affected_rows);
```

## See also

- [SliderTable](SliderTable.md) — name-keyed container referencing these
- [VertexTransform](VertexTransform.md) — the analogous "produces a matrix" node for skinning
