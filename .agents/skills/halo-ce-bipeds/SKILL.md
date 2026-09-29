---
name: halo-ce-bipeds
description: >-
  Detailed reference for the Biped and Unit simulation systems in Halo CE (halo-ce-universal).
  Use whenever modifying characters, player movement, biped physics, ragdolls, health/shields,
  damage processing, or unit animation states.
---

# Halo CE: Biped & Unit System Guide

## 1. Object Inheritance Hierarchy

All physical world entities inherit from `_object_datum`. Halo CE implements single inheritance through struct containment:

```text
struct _object_datum        (Base physical object, transform, health, shields, rendering)
  └── struct _unit_datum     (Controllable entity: inventory, aiming, seats, animations)
        ├── struct _biped_datum   (Walking characters: players, marines, covenants)
        └── struct _vehicle_datum (Drivable vehicles: warthog, ghost, banshee, scorpion)
```

### Full Datum Layouts
- `struct biped_datum`:
  ```c
  struct biped_datum
  {
      long definition_index;
      struct _object_datum object;
      struct _unit_datum unit;
      struct _biped_datum biped;
  };
  ```
- Macro accessors:
  - `biped_get(object_index)`: Returns `(struct biped_datum *)` after verifying `_object_mask_biped`.
  - `unit_get(object_index)`: Returns `(struct unit_datum *)` for bipeds or vehicles.
  - `object_get(object_index)`: Base access for any valid game object.

---

## 2. Biped Data Structure (`source/units/bipeds.h`)

`struct _biped_datum` controls biped-specific locomotion, grounding, and ragdoll state:

```c
struct _biped_datum
{
    unsigned long flags;
    char landing_recovery_counter;
    char landing_recovery_time;
    char state;
    char elevator_ticks;
    long elevator_object_index;
    long support_surface_index;
    long pathfinding_surface_index;
    real_point3d pathfinding_point;
    long last_pathfinding_attempt_time;
    long last_pathfinding_surface_index;
    long impact_target_object_index;
    long last_falling_communication_time;
    long bump_object_index;
    char bump_ticks;
    char airborne_ticks;
    char slipping_ticks;
    char stop_ticks;
    char jump_recovery_timer;
    char player_melee_ticks;
    char player_melee_attack_tick;
    short landing;
    real crouch;                    // 0.0f (standing) to 1.0f (fully crouched)
    real bank;                      // Banking angle during turns or flight
    real_plane3d ground_plane;       // Normal and distance of surface beneath feet
    byte limp_body_current_relaxation_iterations;
    byte limp_body_max_relaxation_iterations;
};
```

### Key Biped Flags
- `_biped_airborne_bit`: Set when character is not supported by ground geometry.
- `_biped_slipping_bit`: Set when character stands on a surface too steep to hold friction.
- `_biped_limp_body_physics_active_bit`: Activated on death or heavy knockdown for limp-body simulation ("ragdoll").

---

## 3. Unit Data Structure (`source/units/units.h`)

`struct _unit_datum` represents the active agent layer driving the object:

### Weapons and Inventory
- `MAXIMUM_WEAPONS_PER_UNIT = 4`: Maximum weapon slots (players use slots 0 and 1).
- `long weapon_object_indices[MAXIMUM_WEAPONS_PER_UNIT]`: Datum indices of carried weapons.
- `short current_weapon_index`: Currently readied weapon slot.
- `short desired_weapon_index`: Pending weapon slot switch.
- `char grenade_counts[NUMBER_OF_UNIT_GRENADE_TYPES]`:
  - `_unit_grenade_human_fragmentation = 0` (Frag grenades)
  - `_unit_grenade_covenant_plasma = 1` (Plasma grenades)

### Facing, Aiming, and Looking Vectors
The unit separates looking direction, weapon aiming direction, and torso facing:
- `real_vector3d desired_facing_vector`: Target forward direction for character body.
- `real_vector3d desired_aiming_vector`: Where the unit wants to point weapon.
- `real_vector3d aiming_vector`: Actual smoothed weapon orientation.
- `real_vector3d aiming_velocity`: Rate of change for aiming vector.
- `real_vector3d desired_looking_vector`: Head/camera direction.
- `real_vector3d throttle`: Movement input in local unit coordinate space (X, Y, Z).

### Vehicle Seating
- `short parent_seat_index`: Slot index in parent vehicle (`NONE` if on foot).
- `long driver_object_index`: Set if this unit is driving another object.
- `long gunner_object_index`: Set if this unit is gunning another object.

### Flashlight & Powerups
- `real integrated_light_power`: Current beam intensity.
- `real integrated_light_battery`: Remaining battery charge (depletes over time).
- `real active_camouflage`: Camouflage value (0.0 = visible, 1.0 = invisible).

---

## 4. Health, Shields, and Damage (`source/objects/objects.h`)

Every unit tracks damage inside `_object_datum`:

```c
real maximum_body_vitality;    // Base health maximum (e.g. 75.0)
real maximum_shield_vitality;  // Base shield maximum (e.g. 75.0)
real body_vitality;            // Current health
real shield_vitality;          // Current shield charge
real current_shield_damage;    // Damage sustained this frame
real current_body_damage;
short shield_stun_ticks;       // Delay counter before shields start recharging
word damage_flags;
```

### Shield Regeneration Cycle
1. When damaged, `shield_stun_ticks` is reset to the damage definition's stun duration.
2. While `shield_stun_ticks > 0`, shields do not recharge.
3. Once ticks expire, `_object_shield_charging_bit` activates, and `shield_vitality` increments per tick up to `maximum_shield_vitality`.
4. Overcharge (Overshield) sets `_object_shield_over_charging_bit`.

---

## 5. Control Pipeline: Input to Biped Update

1. **Input Sampling**: `player_control_update()` reads controller sticks and buttons, populating `struct player_control`.
2. **Unit Control Feed**: `unit_control(unit_index, &control_data)` translates player input into `_unit_datum` fields:
   - Evaluates `UNIT_CONTROL_DRIVER_MASK` vs `UNIT_CONTROL_GUNNER_MASK`.
   - Sets `control_flags` (`_unit_control_jump_bit`, `_unit_control_crouch_modifier_bit`, etc.).
3. **Biped Locomotion Update**: `bipeds_update()`:
   - Raycasts/sweeps physics pill (`biped_get_physics_pill`) against structure BSP.
   - Evaluates ground plane contact (`ground_plane`), applying gravity or friction.
   - Updates `crouch` scalar smoothly towards target crouch state.
   - Evaluates airborne state and handles landing impacts (`landing_recovery_counter`).
4. **Animation Step**: `unit_animation_update()` updates lower-body locomotion cycles and upper-body weapon aiming postures based on current weapon class.
