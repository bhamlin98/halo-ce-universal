---
name: halo-ce-networking
description: >-
  Guide to Halo CE multiplayer networking, system link, and the modern distributed netcode
  in halo-ce-universal. Use when modifying network synchronization, client prediction,
  hit validation, message handlers, lobby management, or port limits (128 players).
---

# Halo CE: Networking & Multiplayer Architecture

## 1. Overview & Dual Netcode Models

Halo CE Universal supports two distinct netcode modes:

1. **Original Lockstep Netcode** (`network.netcode = "lockstep"`):
   - Faithful to the original Xbox 2001 System Link design.
   - Every machine runs on the same strict clock tick (30 Hz).
   - Clients send local input to the host; the host broadcasts all inputs for each tick.
   - All machines execute an identical deterministic simulation.
   - Drawbacks: High latency (one full round-trip delay on inputs) and vulnerable to desync if simulations diverge by a single bit.

2. **Modern Distributed Netcode** (`network.netcode = "distributed"`, default in native builds):
   - Decoupled ticks: Every machine runs on its own local 30 Hz clock without blocking on packet arrivals.
   - **Local Player Prediction**: Clients immediately simulate their own movement and vehicle control locally.
   - **Host Authority**: The host alone determines damage application, entity deaths, object spawns, weapon pickups, and score increments.
   - **State Corrections**: Host sends periodic authoritative entity snapshots. Clients glide mismatched positions smoothly via `render_interpolation.c`.
   - **Shooter-Side Hit Reporting**: Client reports hits on targets; host validates historical position within a tolerance window before applying damage (`port/linux/game/network_damage.c`).

---

## 2. Session Limits: 128 Players (`port/linux/include/halo_port_limits.h`)

While the original Xbox game capped sessions at 16 players across 4 machines, native builds expand capacity:

```c
#define HALO_PORT_MAXIMUM_NETWORK_PLAYERS 128
#define HALO_PORT_MAXIMUM_NETWORK_MACHINES 128
#define HALO_PORT_FD_SETSIZE 256
```

- Player and machine indices are stored as signed 8-bit integers (`0 .. 127`), fitting the 128-player ceiling without breaking internal data structures.
- Extended limits are guarded by `#ifdef HALO_LINUX` / `#ifdef HALO_WINDOWS` to preserve byte-exact matching on the original Xbox MSVC build.

---

## 3. Message Types & Protocol (`source/networking/network_messages.h`)

Communication occurs over UDP ports `0x141E` (server) and `0x141F` (client).

### Key Message Enumerations
```c
enum network_game_message_type
{
    _message_client_broadcast_game_search = 0,
    _message_client_ping,
    _message_server_game_advertise,
    _message_server_pong,
    _message_server_machine_accepted,
    _message_server_machine_rejected,
    _message_server_game_settings_update,
    _message_server_pregame_countdown,
    _message_server_begin_game,
    _message_server_pregame_keep_alive,
    _message_client_join_game_request,
    _message_client_add_player_request_pregame,
    _message_server_game_update,              // Authoritative tick broadcast
    _message_server_add_player_ingame,        // Mid-game player spawn
    _message_server_remove_player_ingame,
    _message_server_game_over,
    _message_client_loaded,
    _message_client_game_update,              // Client input / state upload
    _message_client_add_player_request_ingame
};
```

---

## 4. Join In Progress (JIP) Flow

Distributed netcode allows clients to join ongoing games:
1. **Lobby Handshake**: Client finds game via broadcast advertisement (`_message_server_game_advertise`) carrying `HALO_PORT_NETWORK_VERSION`.
2. **Acceptance & Slot Assignment**: Host accepts machine and allocates a stable datum index in `struct network_game`.
3. **Map Pre-cache & Load**: Host sends settings and current `game_time`. The client loads map tags and initializes local state.
4. **World State Handshake** (`network_objects.c`):
   - Host sends all dynamic objects (vehicles, dropped weapons, powerups).
   - Client maps them to identical datum indices to maintain 1:1 cross-machine references.
   - Client spawns its predicted biped and begins streaming tick updates.

---

## 5. Internet Play & P2P Transport Layer (`port/linux/src/`)

Native builds enable online multiplayer without central dedicated servers:
- **Signaling**: Lightweight MQTT rendezvous for lobby announcements and invite links.
- **NAT Traversal**: STUN requests determine public IP and port mappings; UDP hole punching establishes peer connections.
- **Reliable UDP**: Uses **KCP** (`port/third_party/kcp/`) for TCP-like delivery over UDP for critical RPCs (joins, spawns, score events).
- **Security**: HMAC tokens authenticate packets and protect against spoofed client updates.
