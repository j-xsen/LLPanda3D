# CollisionHandlerGravity

**Source:** `panda/src/collide/collisionHandlerGravity.h` (+ `.I`, `.cxx`)
**Inherits:** [CollisionHandlerPhysical](CollisionHandlerPhysical.md)

Like [CollisionHandlerFloor](CollisionHandlerFloor.md) — snaps to the
highest ray hit each frame — but integrates real vertical velocity and
gravitational acceleration, so it also handles falling, landing impact, and
jumping. The standard handler for a walking/jumping avatar with an actual
downward `CollisionRay`.

## Behavior notes

- **Same "highest hit, capped reach" floor-following as
  [CollisionHandlerFloor](CollisionHandlerFloor.md)**
  (`set_highest_collision()`, `offset`, `reach`), but instead of snapping
  directly, it tracks `_current_velocity` against `_gravity` each frame:
  falls when airborne, applies an impact when landing, and reports
  `is_on_ground()` / `get_airborne_height()` / `get_impact_velocity()`
  afterward.
- **`get_impact_velocity()` is the velocity at the moment of landing** —
  read it right after a frame where `is_on_ground()` transitions to `true`
  to implement fall damage or a landing animation trigger.
- **`add_velocity(v)` accumulates onto the current vertical velocity
  (e.g. for a jump impulse); `set_velocity(v)` replaces it outright.**
- **`max_velocity` caps terminal fall speed**, same role as in
  [CollisionHandlerFloor](CollisionHandlerFloor.md) but specifically bounding
  how fast gravity can accelerate the fall, not just the per-frame Z delta.
- **`set_legacy_mode(true)` switches to an older, less physically-accurate
  integration path** kept for backward compatibility with content tuned
  against the old behavior — new code should leave it `false` (the default).
- **`get_contact_normal()` is the surface normal of the floor last landed
  on** — useful for slope-dependent logic (sliding on steep surfaces, etc.),
  though [CollisionSolid](CollisionSolid.md)'s `effective_normal` mechanism
  can already flatten this to "up" for floor-type solids.

## API

| Signature | Notes |
|---|---|
| `CollisionHandlerGravity()` | |
| `void set_offset(PN_stdfloat)` / `get_offset() const` | Z offset above the highest hit |
| `void set_reach(PN_stdfloat)` / `get_reach() const` | Max distance below the ray origin still considered |
| `void set_gravity(PN_stdfloat)` / `get_gravity() const` | Downward acceleration |
| `void set_velocity(PN_stdfloat)` / `void add_velocity(PN_stdfloat)` / `PN_stdfloat get_velocity() const` | Current vertical velocity; set replaces, add accumulates (e.g. jump impulse) |
| `void set_max_velocity(PN_stdfloat)` / `get_max_velocity() const` | Terminal fall speed cap |
| `PN_stdfloat get_airborne_height() const` | Distance above the last known floor |
| `bool is_on_ground() const` | |
| `PN_stdfloat get_impact_velocity() const` | Velocity at moment of landing |
| `const LVector3 &get_contact_normal() const` | Normal of the floor last landed on |
| `void set_legacy_mode(bool)` / `bool get_legacy_mode() const` | Older integration path for compatibility |

## Usage

```cpp
PT(CollisionHandlerGravity) gravity = new CollisionHandlerGravity();
gravity->set_gravity(32.0);
gravity->add_collider(feet_ray_np, avatar_np);
ctrav.add_collider(feet_ray_np, gravity);

// on jump input:
if (gravity->is_on_ground()) {
  gravity->add_velocity(jump_speed);
}
```

## See also

[CollisionHandlerFloor.md](CollisionHandlerFloor.md) · [CollisionHandlerPhysical.md](CollisionHandlerPhysical.md)
· [README.md](README.md)
