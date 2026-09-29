---
name: halo-ce-input-interaction
description: >-
  Guide to input handling, SDL3 device mapping, direct mouse look, and player interaction
  systems (vehicle boarding, weapon swap, switches, devices) in halo-ce-universal.
  Use when modifying controller mapping, mouse sensitivity, use/action button logic, or device mechanics.
---

# Halo CE: Input & Interaction System Guide

## 1. Input Processing Architecture

Input flows from the hardware platform layer down to game simulation through a structured pipeline:

```text
[Hardware Devices] (SDL3 Gamepads, Keyboard, Raw Mouse)
       │
       ▼
[Platform Layer] (`port/linux/src/xinput_sdl.c`)
  - Emulates Xbox XInput controller states for Ports 0..3
  - Exposes `halo_linux_mouse_look()` for direct mouse delta injection
       │
       ▼
[Input Abstraction] (`source/input/input_abstraction.c`)
  - Reads button state, applies deadzones, invert look, sensitivity
  - Outputs `struct game_input_state`
       │
       ▼
[Player Control] (`source/game/player_control.c`)
  - Maps input state to `struct player_control` for each local player
  - Applies auto-aim, magnetism, and camera limits
       │
       ▼
[Unit Control] (`source/units/units.c`)
  - Injects `control_flags` and vectors into `_unit_datum`
```

---

## 2. Modern Input & Mouse Handling (`port/linux/src/xinput_sdl.c`)

### Port 0 Multiplexing
- **Port 0** combines keyboard and mouse inputs merged with the first connected gamepad.
- Additional connected gamepads map to Ports 1, 2, and 3 for local split-screen players.

### Keyboard & Mouse Defaults
| Input | Xbox Equivalent | Action |
|---|---|---|
| `W`, `A`, `S`, `D` | Left Thumbstick | Movement (Throttle) |
| Mouse Motion | Direct Aim | Direct yaw/pitch delta (bypasses analog stick rate) |
| Left Mouse Button | Right Trigger | Primary Fire |
| Right Mouse Button, `G` | Left Trigger | Throw Grenade |
| `Space`, `Enter` | `A` Button | Jump |
| `F`, `Backspace`, `Mouse 4` | `B` Button | Melee Attack |
| `E`, `R` | `X` Button | Action / Reload |
| `Tab`, `Mouse Wheel` | `Y` Button | Switch Weapon |
| `Q` | White Button | Toggle Flashlight |
| `Left Ctrl`, `C` | Left Stick Click | Crouch |
| `Z`, `Middle Mouse` | Right Stick Click | Zoom Scope |
| `Escape` | Start Button | Pause Menu |
| `` ` `` (Backquote) | Debug Console | Open/close command line |
| `F12` | Window Manager | Recapture / release mouse pointer |

### Direct Mouse Look (`halo_linux_mouse_look`)
Unlike thumbsticks which produce velocity (rate-based turning), mouse input produces raw angular displacements:
- The game's look code directly queries `halo_linux_mouse_look(gamepad_index, &yaw, &pitch)`.
- Displacements are added directly to the camera facing angle: `scale = 0.0022f * sensitivity`.

---

## 3. World Interaction System (`source/units/units.c`, `source/devices/`)

When a player holds or taps the Action (`X`) button (`_player_control_action_bit`), the game scans nearby entities within interaction radius.

### Priority Resolution Order
1. **Vehicle Entry**: Checked first if an enterable vehicle is within range.
2. **Item / Weapon Swap**: Checked next if a weapon or item is in proximity.
3. **Device Activation**: Checked for switches, control panels, or manual doors.

---

## 4. Vehicle Entry & Seat Selection (`source/units/units.c`)

When evaluating vehicle interaction around a player:
1. `unit_find_nearby_seat()` iterates through all seats in the target vehicle's definition.
2. Checks seat qualification:
   - Does seat require a driver (`_unit_seat_requires_driver_bit`)?
   - Is seat empty, or occupied by an AI that can be evicted (`ai_try_vehicle_eviction`)?
3. **Driver Preference Weighting**:
   - Driver seats are prioritized: non-driver seat distances are multiplied by `1.5f` so the player automatically boards the driver seat if equidistant.
4. **Transition**: If approved, `unit_enter_seat(unit_index, vehicle_index, seat_index)` attaches the player to the vehicle marker node and transfers steering authority to `_unit_control_driver_bit`.

---

## 5. Device Controls & Machines (`source/devices/`)

Interactive scenery uses the Device system (`_object_mask_device`):
- **`device_machines.c`** (`'mach'` tag): Moving objects like automated doors, blast doors, and elevators.
  - Can operate automatically via trigger proximity, or manually via switches.
  - Supports opening by melee attack (`_machine_opened_by_melee_attack_bit`).
- **`device_controls.c`** (`'ctrl'` tag): Interactive panels and switches.
  - Triggers linked power groups or script callbacks when the player interacts.
  - Supports single-use or reversible toggling (`_scenario_device_changes_only_once_bit`).
