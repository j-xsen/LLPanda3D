# Material

**Source:** `panda/src/gobj/material.h` (+ `.I`, `.cxx`)
**Inherits:** TypedWritableReferenceCount, Namable **Inherited by:** (none)

Defines how an object appears under lighting — only relevant when
lighting is enabled (`LightAttrib`) and a `MaterialAttrib` applies this
material to a subgraph (both in `pgraph`, see
[../pgraph/README.md](../pgraph/README.md)). Two independent workflows are
supported on the same class: the "classic" ambient/diffuse/specular/
emission color workflow, and a "metalness" (PBR-ish) workflow of a single
`base_color` plus a `metallic` scalar, from which effective ambient/
diffuse/specular colors are algebraically derived. Highlight size comes
from either `shininess` (specular exponent) or a perceptually-linear
`roughness` in `[0,1]`.

## Behavior notes

- **Every color field has an independent "has it been set" flag**
  (`F_ambient`/`F_diffuse`/`F_specular`/`F_emission`/`F_base_color` in the
  `_flags` bitmask, queried via `has_ambient()` etc.). Unset fields fall
  back to sensible defaults at the *shader-generation* level (e.g. "use
  the object/vertex color") — the getters (`get_ambient()` etc.) still
  return a concrete stored value even when unset, they just return
  whatever the last recompute left there, not a sentinel.
- **`set_base_color()`/`set_metallic()` recompute the classic fields.**
  Calling `set_base_color()` sets `F_base_color | F_metallic`, clears
  `F_ambient | F_diffuse | F_specular`, and recomputes `_ambient` (=
  base color), `_diffuse` (= `base_color * (1 - metallic)`), and
  `_specular` (Fresnel-`f0`-from-`refractive_index` blended with
  `base_color * metallic`) — so setting base color silently discards any
  previously explicit ambient/diffuse/specular you'd set. The two
  workflows are not meant to be mixed field-by-field.
- **`get_base_color()` falls back to `_diffuse`** when neither base color
  nor metallic has been explicitly set — so reading base color on a
  classic-workflow material returns its diffuse color rather than black.
- **`clear_ambient()`/`clear_diffuse()`** don't reset to a neutral
  default — they recompute from `_base_color` (`clear_ambient()` sets
  `_ambient = _base_color`; `clear_diffuse()` sets
  `_diffuse = _base_color * (1 - _metallic)`), i.e. "clear" means
  "fall back to the metalness-workflow-derived value," not "zero it out."
- **Likely bug:** `is_used_by_auto_shader()` — the internal check every
  setter uses to decide whether to call
  `GraphicsStateGuardianBase::mark_rehash_generated_shaders()` — tests
  `_flags & F_attrib_lock` (`0x040`), while the only place that flag is
  actually *set* is `mark_used_by_auto_shader()`, which sets a different
  bit, `F_used_by_auto_shader` (`0x800`). `is_attrib_locked()` also reads
  `F_attrib_lock` (and per its own docstring is a dead/deprecated-since-
  1.10 API). Net effect: `mark_used_by_auto_shader()` never actually
  causes `is_used_by_auto_shader()` to return true, so a mutation on a
  `Material` already consumed by the shader generator may not trigger a
  shader regeneration through this path — check for `set_attrib_lock()`
  being called elsewhere (e.g. from the shader generator itself) if you
  hit stale-generated-shader symptoms after mutating a live `Material`.
- **`get_default()`** lazily allocates one shared default `Material`
  named `"default"` on first call and reuses it — comparable to
  `SamplerState::get_default()`.
- Copy-assignment (`operator=`) explicitly preserves the *target's*
  `F_attrib_lock`/`F_used_by_auto_shader` bits rather than copying the
  source's, while the copy-constructor strips both from the copy —
  auto-shader bookkeeping is per-instance-in-use, not part of a
  material's logical value.

## API

**Classic workflow (ambient/diffuse/specular/emission):**

| Signature | Notes |
|---|---|
| `bool has_ambient/diffuse/specular/emission() const` | Explicitly-set check. |
| `const LColor &get_ambient/diffuse/specular/emission() const` | Current value (set or derived). |
| `void set_ambient/diffuse/specular/emission(LColor)` | Sets the value and the corresponding `has_*` flag. |
| `void clear_ambient/diffuse/specular/emission()` | Unsets the `has_*` flag; see recompute behavior above for ambient/diffuse. |

**Metalness workflow:**

| Signature | Notes |
|---|---|
| `bool has_base_color() const` / `const LColor &get_base_color() const` | See fallback-to-diffuse note above. |
| `void set_base_color(LColor)` / `void clear_base_color()` | Recomputes ambient/diffuse/specular — see notes. |
| `bool has_metallic() const` / `PN_stdfloat get_metallic() const` | 0 (dielectric) if unset. |
| `void set_metallic(PN_stdfloat)` / `void clear_metallic()` | |
| `bool has_refractive_index() const` / `PN_stdfloat get_refractive_index() const` | 1 (no reflection) if unset; feeds the Fresnel `f0` term above. |
| `void set_refractive_index(PN_stdfloat)` | |

**Highlight shape:**

| Signature | Notes |
|---|---|
| `PN_stdfloat get_shininess() const` / `void set_shininess(PN_stdfloat)` | Classic specular exponent. |
| `bool has_roughness() const` / `PN_stdfloat get_roughness() const` / `void set_roughness(PN_stdfloat)` | Perceptually-linear `[0,1]` alternative; default 1. |

**Lighting flags:**

| Signature | Notes |
|---|---|
| `bool get_local() const` / `void set_local(bool)` | Camera-relative vs. orthogonal specular highlights; default true. |
| `bool get_twoside() const` / `void set_twoside(bool)` | Light both polygon faces; default false. |

**Comparison / misc:**

| Signature | Notes |
|---|---|
| `int compare_to(const Material &) const` / `operator==/!=/<` | Value comparison. |
| `static Material *get_default()` | Shared lazily-created default instance. |
| `void output(ostream &) const` / `void write(ostream &, int indent) const` | Debug printing. |

## Usage

```cpp
PT(Material) mat = new Material("shiny");
mat->set_ambient(LColor(0.2, 0.2, 0.2, 1));
mat->set_diffuse(LColor(0.8, 0.1, 0.1, 1));
mat->set_shininess(32.0);
node_path.set_material(mat);  // pgraph, applies a MaterialAttrib
```

## See also

- `MaterialAttrib`, `LightAttrib` — `pgraph`
  ([../pgraph/README.md](../pgraph/README.md))
- [MaterialPool](MaterialPool.md) — name-keyed cache for shared `Material`
  instances
