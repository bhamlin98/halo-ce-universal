# AGENTS.md — Guidelines for halo-ce-universal

This repository is a clean-room decompilation and native port of **Halo: Combat Evolved for Xbox (build 2342, XDK 3911, VC7 13.00.9254.1)** running natively on Linux, Windows, and Android.

---

## 1. Project Skills

Specialized skill manuals are maintained under `.agents/skills/`. **You must consult the relevant skill before writing or modifying code in these domains:**

- **Overall Architecture & Build System**: [`.agents/skills/halo-ce-architecture/SKILL.md`](.agents/skills/halo-ce-architecture/SKILL.md)  
  *Covers source layout, memory arrays, datum handles, build configurations, and campaign workflow.*
- **Biped & Unit Simulation**: [`.agents/skills/halo-ce-bipeds/SKILL.md`](.agents/skills/halo-ce-bipeds/SKILL.md)  
  *Covers character physics, health/shields, biped states, limp-body relaxation, and animation updates.*
- **Weapons, Items, & Vehicles**: [`.agents/skills/halo-ce-weapons-vehicles/SKILL.md`](.agents/skills/halo-ce-weapons-vehicles/SKILL.md)  
  *Covers triggers, magazines, projectile physics, dropped items, vehicle seating, and suspension.*
- **Distributed Netcode & System Link**: [`.agents/skills/halo-ce-networking/SKILL.md`](.agents/skills/halo-ce-networking/SKILL.md)  
  *Covers lockstep vs. distributed netcode, 128-player limits, host authority, and P2P UDP transport.*
- **Input & World Interaction**: [`.agents/skills/halo-ce-input-interaction/SKILL.md`](.agents/skills/halo-ce-input-interaction/SKILL.md)  
  *Covers direct mouse look, SDL3 controller mapping, action button priorities, and device machines/controls.*

---

## 2. Core Architectural Invariants

1. **Strict C89 / GNU89 Style & CSeries Primitives**:
   - Use engine types: `real` (float), `boolean` (`TRUE`/`FALSE`), `byte`, `word`, `tag` (FourCC).
   - Use `NONE` (-1) as the sentinel for invalid indices or handles.
   - Always route CRT calls through `cseries.h` wrappers (`csmemcpy`, `csmemset`, `csstrcmp`, `csstrlen`, `cssprintf`). Never introduce raw `<string.h>` or `<stdlib.h>` into game source files.
2. **Deterministic Math for Multiplayer**:
   - Native builds must maintain floating-point lockstep parity across Linux, Windows, and Android.
   - Floating-point contraction is disabled (`-ffp-contract=off`). Math functions must use `halo_math.h` (`musl-math`).
3. **Assertions & Matching Standard**:
   - Preserve original source locations using `match_assert(file, line, expr)`.
   - In non-matching/native builds, assertions remain enabled in debug mode and are elided in `HALO_RELEASE`.
   - Never use fake or byte-steering tricks that compromise source readability or authenticity (`docs/matching_methodology.md`).
4. **Campaign & Tooling Rules**:
   - When matching functions, use `python tools/campaign/gate.py --fn <symbol>` to test isolated compilation against target COFF sections without corrupting `build/`.
   - Parked functions in `config/parked.json` are honest fuzzy records and receive zero completion credit.
