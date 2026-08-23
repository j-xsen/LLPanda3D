# VertexTransform

**Source:** `panda/src/gobj/vertexTransform.h` (+ `.I`, `.cxx`)
**Inherits:** `TypedWritableReferenceCount` **Inherited by:** `UserVertexTransform` (see below); `JointVertexTransform` (in `chan`, undocumented)

Abstract base for "produces a 4x4 matrix, recomputed on demand" nodes used
to implement soft-skinned and hardware-skinned vertex animation. Concrete
subclasses define *how* the matrix is computed — `UserVertexTransform`
(this module, folded in below) just holds a constant app-supplied matrix;
`JointVertexTransform` (in `panda/src/chan`, undocumented) derives it from
an animated skeleton joint, which is the actual common case in loaded
character models. A `VertexTransform` is referenced from either a
[TransformTable](TransformTable.md) (hardware skinning) or, wrapped in a
[TransformBlend](TransformBlend.md), a
[TransformBlendTable](TransformBlendTable.md) (CPU skinning).

## Behavior notes

- Pure virtual `get_matrix(LMatrix4 &matrix) const = 0` is the only
  required override; `mult_matrix()` (premultiply with a previous matrix)
  and `accumulate_matrix()` (weighted-add into an accumulator, used by
  `TransformBlend`'s blending) both have default implementations built on
  top of `get_matrix()`, but a subclass may override them for efficiency
  if it can avoid materializing the full matrix.
- **Global modification sequence:** every `VertexTransform` in the process
  shares one monotonic counter (`get_next_modified()`/
  `get_global_modified()`, backed by a static `PipelineCycler`). A
  subclass must call the protected `mark_modified(Thread*)` whenever its
  computed matrix changes; this both bumps the object's own `_modified`
  stamp to a freshly-allocated global sequence number *and* immediately
  notifies every [TransformTable](TransformTable.md) currently holding a
  back-pointer to it (`_tables`, populated by `TransformTable::
  register_table()`) via `update_modified()`. This is what lets
  `TransformBlend::update_blend()` cheaply detect staleness by comparing a
  single sequence number instead of re-checking every contributing
  transform's individual state.
- The destructor asserts `_tables.empty()` — a `VertexTransform` must be
  removed from every registered `TransformTable` before it can be
  destroyed; this is enforced automatically since `TransformTable`
  unregisters (or the table itself is destroyed) before releasing its
  `CPT(VertexTransform)` references, but a bug that keeps a table
  registered past the transform's expected lifetime will trip this assert.

## API

| Method | Notes |
|---|---|
| `virtual get_matrix(LMatrix4 &matrix) const = 0` | The core override every subclass must supply. |
| `virtual mult_matrix(LMatrix4 &result, const LMatrix4 &previous) const` | `result = get_matrix() * previous`; default impl calls `get_matrix()`. |
| `virtual accumulate_matrix(LMatrix4 &accum, PN_stdfloat weight) const` | `accum += get_matrix() * weight`; default impl calls `get_matrix()`. |
| `get_modified(Thread*)` | This transform's last-changed sequence number. |
| `static get_next_modified(Thread*)` | Allocate and return the next global sequence number. |
| `static get_global_modified(Thread*)` | Current global sequence number (for staleness comparison). |
| `mark_modified(Thread*)` *(protected)* | Subclasses call this after any change to their computed matrix. |
| `output(ostream&)` / `write(ostream&, indent_level)` | Debug dump; `write()` also prints the resolved matrix. |

## `UserVertexTransform` (folded subclass)

**Source:** `panda/src/gobj/userVertexTransform.h` (+ `.I`, `.cxx`)

A trivial `VertexTransform` subclass holding an app-supplied constant
`LMatrix4`, set via `set_matrix()`. The header notes it's "rarely used
except for testing" — real character skinning uses `JointVertexTransform`
(`chan`, undocumented) instead, which derives its matrix from an animated
joint. `set_matrix()` calls `mark_modified()` internally so dependent
`TransformTable`s/`TransformBlend`s see the update automatically.

| Method | Notes |
|---|---|
| `UserVertexTransform(const std::string &name)` | Name is cosmetic (debug output only). |
| `get_name()` | |
| `set_matrix(const LMatrix4 &)` | Replace the constant matrix; triggers `mark_modified()`. |
| `get_matrix(LMatrix4 &) const` (override) | Returns the stored matrix. |

## Usage

```cpp
PT(UserVertexTransform) t = new UserVertexTransform("manual");
t->set_matrix(LMatrix4::translate_mat(0, 0, 1));
PT(TransformTable) table = new TransformTable;
table->add_transform(t);
```

## See also

- [TransformTable](TransformTable.md) — GPU-palette container referencing these
- [TransformBlend](TransformBlend.md), [TransformBlendTable](TransformBlendTable.md) — CPU-blending container
- [VertexSlider](VertexSlider.md) — the analogous "produces a float" node for morphs
