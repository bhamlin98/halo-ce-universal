---
name: halo-ce-architecture
description: >-
  Comprehensive guide to the Halo: Combat Evolved decompilation and port (halo-ce-universal).
  Use whenever working on build system, decompilation campaign, codebase structure, memory
  arrays, cseries primitives, or port layers (Linux, Windows, Android).
---

# Halo CE Universal: Architecture and Decompilation Guide

## 1. Project Overview

`halo-ce-universal` is a clean-room decompilation and native port of **Halo: Combat Evolved for Xbox (build 2342, "January 2002" / XDK 3911, VC7 13.00.9254.1)**.
- **Target ROM**: `cachebeta.exe` (SHA-256: `4cc87b45f721270392a96f1674ed2b5cd4a7bb4355faeab4531d1cf1884d9520`)
- **Native Port Targets**: 32-bit x86 Linux (ELF), 32-bit x86 Windows (PE), and ARM64 Android (APK/ELF).
- **Core Philosophy**:
  1. *Byte-exact decompilation* against January 2002 Xbox COFF objects.
  2. *Single unified codebase* where the same game source compiles for both exact matching (VC7) and modern native execution (Clang + SDL3 + OpenGL).

---

## 2. Directory and Source Module Layout

The codebase under `source/` is structured into 37 distinct submodules following Bungie's original layout:

| Subdirectory | Core Responsibility |
|---|---|
| `ai/` | Full AI system: actors, commands, encounters, props, pathfinding (`path_obstacles`, `structure_bsp`), actor types (grunt, elite, flood, marine...) |
| `bitmaps/` | Bitmap formats, drawing, swizzling, and decompression macros |
| `cseries/` | **Engine foundation**: primitive types, error handling, assertions, memory, profiling, string utilities |
| `devices/` | World interactive machinery: device machines (doors, elevators), controls (switches, buttons), light fixtures |
| `effects/` | Particle systems, decals, contrails, weather systems |
| `game/` | Game loop, game variants (CTF, Slayer, King, Oddball, Race), player records, `player_control`, game time, aim assist |
| `hs/` | HaloScript bytecode compiler and virtual machine |
| `input/` | Controller abstraction, input preferences, device polling |
| `interface/` | HUD, user interface, menus, fonts, UI widgets |
| `items/` | Weapons, items, equipment, projectiles, garbage collection |
| `main/` | Application entrypoint, initialization sequence, main loop, debug console |
| `math/` | 2D/3D vectors, matrices, quaternions, planes, Euler angles (`real_math.h`, `integer_math.h`) |
| `memory/` | `data_array`, hash tables, LRU caches, memory pools, circular queues, packet encoding |
| `models/` | 3D model structures, node matrices, LOD definitions |
| `networking/` | Network game managers, client/server message handlers, packet serialization, telnet console |
| `objects/` | Universal object header and datum (`_object_datum`), damage pipeline, placement, attachments |
| `physics/` | Collision detection, rigid-body dynamics, Havok integration points |
| `rasterizer/` | D3D8 / NV2A draw calls, vertex/pixel shaders, transparent geometry, dynamic lights, environment maps |
| `render/` | High-level render pipeline, cameras, view frustums, debug visualization |
| `scenario/` | Map/level structures, scenario objects, encounter placements, trigger volumes |
| `sound/` | Audio engine, sound tags, DirectSound integration |
| `structures/` | Structure BSP (world collision geometry, portals, lightmaps, clusters) |
| `tag_files/` | Bungie Tag system (`tag_files.h`), tag groups, tag memory loading (`tag_get`) |
| `units/` | Unit base class, bipeds, vehicles, animation controller, weapon inventory |

---

## 3. Foundational Types & CSeries Primitives (`source/cseries/cseries.h`)

All modules depend on `source/cseries/cseries.h`. Never introduce standard CRT headers directly into game source files.

### Fundamental Types
```c
typedef unsigned char byte;
typedef unsigned short word;
typedef float real;              // 32-bit IEEE float; always append 'f' to constants
typedef byte boolean;            // TRUE (1) or FALSE (0)
typedef unsigned long tag;       // 4-byte FourCC tag class (e.g. 'bipd', 'weap', 'vehi')
```

### Core Constants & Macros
- `NONE = -1`: Canonical sentinel for invalid datum index, object index, or null handle.
- `TICKS_PER_SECOND = 30`: Standard game simulation rate (33.333 ms per tick).
- `VBLANKS_PER_SECOND = 60`: Standard display refresh target.
- Bitwise macros: `FLAG(bit)`, `TEST_FLAG(flags, bit)`, `SET_FLAG(flags, bit, value)`.
- `cs*` CRT wrappers: `csmemcpy`, `csmemset`, `csstrcmp`, `csstrlen`, `cssprintf`.

### Assertions (`docs/assertions.md`)
- `assert(expr)` / `dassert(expr, diagnostic)`: Standard assertion.
- `match_assert(file, line, expr)`: Preserves original Xbox source file and line number for byte-exact builds.
- `vassert(expr, csprintf(temporary, format, ...))`: Assertion with formatting. Note: VC7 lacks `__VA_ARGS__`, so `temporary[256]` is used.
- In `HALO_RELEASE` builds, assertions are elided to `(void)(expr)`.

---

## 4. Memory Architecture & Datum Handles

Halo CE relies heavily on the **`data_array`** pattern for all dynamic objects (`source/memory/data.h`).

### Datum Index Structure
A `datum_index` (or `long object_index`) is a 32-bit packed integer:
- **Low 16 bits**: Absolute index into the array (`0 .. maximum_count - 1`).
- **High 16 bits**: Salt (generation counter) to detect stale references.
- Conversion macros:
  - `DATUM_INDEX_TO_ABSOLUTE_INDEX(index)` -> extracts array slot.
  - `DATUM_INDEX_TO_IDENTIFIER(index)` -> extracts salt.
  - `DATUM_INDEX_NEW(slot, salt)` -> constructs handle.

---

## 5. Decompilation Campaign & Matching Workflow

The project uses a strict, evidence-based matching methodology (`docs/matching_methodology.md`):

### Campaign Tools (`tools/campaign/`)
- `python tools/campaign/gate.py --fn <symbol>`: Compiles a single translation unit and tests against the target binary section without modifying `build/`.
- `python tools/campaign/board.py`: Displays the overall completion scoreboard (Exact, Fuzzy, Unwritten).
- `python tools/campaign/alndiff.py`: Shows side-by-side assembly diffs with normalized relocations.
- `python tools/campaign/tinfo.py`: Inspects section sizes, hashes, and COFF relocations.

### Config Registry (`config/`)
- `config/config.json`: Master build configuration containing object statuses (`MISSING`, `NonMatching`, `Matching`).
- `config/symbols.json`: Maps target executable offsets to symbol names and static/extern linkage.
- `config/splits.json`: Defines section boundaries (`.text`, `D3D`, `DSOUND`, `XNET`, etc.).
- `config/parked.json`: Documents unmatchable functions with detailed evidence (Class A-F). Parked functions receive **zero** completion credit.
- `config/semantic_matches.json`: Allowlist for functions confirmed byte-exact except for compiler-internal label artifacting.

---

## 6. Build System & Platform Port Layer

### Build Pipeline
1. `python configure.py`: Generates `build.ninja` and `objdiff.json`.
   - `--release`: Strips assertions.
   - `--portable`: Target generic x86-64 / ARM64 instead of native host CPU.
2. `ninja`: Default target compiles the matching build via wibo/wine on Linux.
   - `ninja windows`: Builds native Windows binary `build/windows/halo.exe`.
   - `ninja linux`: Builds native Linux binary `build/linux/halo`.
   - `ninja android_apk`: Assembles complete Android package.

### Native Port Architecture
- **`port/include/xdk/`**: Clean-room implementation of Xbox SDK headers (`xdk_d3d8.h`, `xdk_dsound.h`, `xtl.h`).
- **`port/linux/src/`**: Shared implementation for POSIX & Windows:
  - `d3d8_gl.c`: Direct3D 8 / NV2A register combiners translated to OpenGL 4.5 GLSL.
  - `dsound_sdl.c`: DirectSound mixing to SDL3 audio streams.
  - `xinput_sdl.c`: Modern gamepad, mouse, and keyboard input mapped to XInput.
  - `xbox_memory.c`: Enforces virtual memory allocation at `0x80000000` to satisfy fixed Xbox pointers.
  - `port/third_party/musl-math/`: Deterministic floating-point math across all architectures. Always compile with `-ffp-contract=off`.
