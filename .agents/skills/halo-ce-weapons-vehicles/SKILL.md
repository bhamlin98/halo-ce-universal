---
name: halo-ce-weapons-vehicles
description: >-
  Technical reference for Weapons, Items, Projectiles, and Vehicles in Halo CE (halo-ce-universal).
  Use whenever working on weapon triggers, magazines, ammunition, reloading, heat/battery drain,
  first-person animations, vehicle physics, seats, or projectile ballistics.
---

# Halo CE: Weapons, Items, Projectiles & Vehicles Guide

## 1. Weapons System (`source/items/weapons.h`)

Weapons are derived from items (`struct weapon_datum` contains `_object_datum`, `_item_datum`, and `_weapon_datum`):

```c
struct _weapon_datum
{
    unsigned long flags;
    word control_flags;
    real primary_trigger;                  // Trigger pull depth (0.0 to 1.0)
    char state;                            // Current weapon lifecycle state
    char last_reported_state;
    short state_timer;                     // Ticks remaining in current state
    real heat;                             // Overheat accumulation (0.0 to 1.0)
    real age;
    real overcharged;                      // Plasma pistol charge accumulation
    real integrated_light_power;
    char integrated_light_delay_ticks;
    long tracked_object_index;             // Rocket launcher lock-on target
    real recoil_angular_velocity;
    short recoil_recovery_time;
    short shots_until_demotion;
    short alternate_shots_loaded;
    struct weapon_trigger triggers[2];     // Primary and secondary triggers
    struct weapon_magazine magazines[2];   // Primary and secondary ammo pools
    struct animation_state animation;
    long overheated_effect_index;
    long game_time_last_fired;
};
```

### Weapon States (`_weapon_state_*`)
- `_weapon_state_idle`: Ready to fire.
- `_weapon_state_primary_recoil` / `secondary_recoil`: Weapon in kickback animation.
- `_weapon_state_primary_chamber` / `secondary_chamber`: Cycling round into chamber.
- `_weapon_state_primary_reload` / `secondary_reload`: Active reload animation.
- `_weapon_state_primary_charged` / `secondary_charged`: Plasma weapon charged state.
- `_weapon_state_ready`: Transitioning into ready posture.
- `_weapon_state_put_away`: Holstering weapon during weapon swap.

### Triggers & Magazines
- **`struct weapon_trigger`**: Tracks `rate_of_fire`, `error` (bloom), `firing_effect_shots_remaining`, and state (`_trigger_charging`, `_trigger_recovering`, `_trigger_spewing`).
- **`struct weapon_magazine`**: Tracks `rounds_loaded`, `rounds_total` (reserve ammo), and `state` (`_magazine_idle`, `_magazine_reloading`, `_magazine_chambering`).

---

## 2. Items & Equipment (`source/items/items.h`)

`struct _item_datum` provides resting physics for dropped weapons, grenades, and powerups:

```c
struct _item_datum
{
    unsigned long flags;
    short detonation_ticks;             // Ticks until explosive arm/detonation
    short rested_surface_index;         // Surface geometry index the item rests on
    short bsp_index;
    long ignore_object_index;           // Avoid colliding with dropper temporarily
    long last_owned_time;
    long item_on_rest_object_index;     // Object (e.g. vehicle) item is resting on
    real_point3d item_rest_object_offset;
    real_vector3d rotation_axis;
    real rotation_sine;
    real rotation_cosine;
};
```

- When picked up by a unit, an item enters the unit's inventory via `item_in_unit_inventory(item_index, unit_index)`.
- It is unlinked from the world collision hierarchy until dropped or discarded.

---

## 3. Projectiles (`source/items/projectiles.h`)

Projectiles represent simulated ballistic or energy rounds (e.g. bullets, rockets, plasma bolts, needles):
- Tracked via `struct projectile_datum` (inherits from `_object_datum`).
- Updated each tick via `projectiles_update()`, performing line-segment sweeps through the structure BSP and object clusters.
- Material interactions trigger tag-defined effects (e.g., sparks, ricochet sounds, or explosions) and apply damage via `damage_apply_to_object()`.

---

## 4. Vehicles System (`source/units/vehicles.h`, `source/units/vehicle_datum.h`)

Vehicles derive from units (`struct vehicle_datum` contains `_object_datum`, `_unit_datum`, and `_vehicle_datum`):

```c
struct _vehicle_datum
{
    word flags;
    short stop_time;
    byte airborne_ticks;
    byte upending_type;
    byte upending_ticks;
    byte on_ground_ticks;
    real speed;                         // Forward velocity scalar
    real slide;                         // Lateral sliding velocity
    real turn;                          // Current steering angle
    real wheel;                         // Wheel rotation angle
    real left_tread;                    // Scorpion tank tread animation
    real right_tread;
    real hover;                         // Ghost / Banshee hover engine thrust
    real thrust;
    byte suspension[8];                 // Per-wheel suspension compression
    real_point3d hover_position;
    real_vector3d collision_force;
    real_vector3d collision_torque;
    unsigned long stuck_mass_point_flags;
};
```

### Seating Architecture
Defined in the vehicle tag definition (`struct unit_seat`):
- `_unit_seat_driver_bit`: Unit controls steering, throttle, and forward weapons.
- `_unit_seat_gunner_bit`: Unit controls turret orientation and heavy armament.
- Passenger seats: Ride-along positions allowing weapon usage or fixed poses.
- Occupant attachment: Unit's position and orientation are locked to marker attachments on vehicle nodes, with damage passthrough or shields calculated per tag rules.
