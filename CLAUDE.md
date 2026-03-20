# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

DayZ server mission and loot economy configuration for Chernarus+ (offline/local server). This is a DayZ modding project — not a traditional software project with build/test/lint tooling. Changes are validated by running a DayZ server with these files.

## Repository Structure

- **`dayzOffline.chernarusplus/`** — The active mission folder deployed to a DayZ server. This is the working copy where modifications are made.
- **`extracted-game-data/`** — Vanilla (unmodified) game data extracted for reference. Contains data for three maps: `chernarusplus`, `enoch` (Livonia), and `sakhal`, plus shared `scripts/`. **Do not modify files here** — this is the baseline for diffing against custom changes.
- **`lootTiers.jpg`** — Visual reference map showing the four loot tier zones on Chernarus+.

## Key Configuration Files (in `dayzOffline.chernarusplus/`)

### Loot Economy (Central Economy / CE)
- **`db/types.xml`** — Master item spawn definitions. Each `<type>` entry controls: nominal (target count on map), min, lifetime, restock timer, spawn flags, category, usage zones, and tier values. This is the primary file for loot balance tuning.
- **`cfglimitsdefinition.xml`** — Defines valid categories (tools, weapons, food, etc.), tags (floor, shelves, ground), usage flags (Military, Police, Farm, etc.), and value flags (Tier1–Tier4, Unique).
- **`cfglimitsdefinitionuser.xml`** — User-defined extensions to limits definitions.
- **`cfgspawnabletypes.xml`** — Controls item attachment/cargo randomization (e.g., which attachments spawn on weapons).
- **`cfgrandompresets.xml`** — Defines random preset groups for spawnable type cargo/attachments.
- **`cfgeconomycore.xml`** — Root classes, CE defaults, and file includes for the economy system.
- **`db/economy.xml`** — CE file include paths and spawn area definitions.
- **`db/globals.xml`** — Global economy parameters (cleanup timers, spawn rates, animal counts, etc.).

### Events & Environment
- **`db/events.xml`** — Dynamic event definitions (vehicle spawns, helicopter crashes, animal herds, infected hordes).
- **`cfgeventgroups.xml`** / **`cfgeventspawns.xml`** — Event grouping and spawn point data.
- **`env/`** — Animal territory XMLs (bear, wolf, deer, zombie territories, etc.).
- **`cfgeffectarea.json`** — Contaminated zone definitions.
- **`cfgweather.xml`** — Weather cycle configuration.

### Mission
- **`init.c`** — Mission entry point (Enforce script). Initializes the CE, sets date, defines `CustomMission` class with player spawn loadout logic.
- **`cfggameplay.json`** — Gameplay parameter overrides.
- **`cfgplayerspawnpoints.xml`** — Player spawn locations.
- **`mapgroupcluster*.xml`** / **`mapgroupproto.xml`** — Building/loot point definitions mapping buildings to spawn positions.

### Scripts
- **`scripts/`** — Enforce script (.c) overrides organized by engine layers: `1_core/`, `3_game/`, `4_world/`, `5_mission/`. These override vanilla game scripts for custom behavior (items, recipes, vehicles, UI, etc.).

## DayZ Economy Concepts

- **Nominal/Min**: Target and minimum item counts on the map. CE spawns items to maintain nominal, respawns when count drops below min.
- **Lifetime**: Seconds before an untouched item despawns.
- **Restock**: Minimum seconds between respawn attempts for an item type.
- **Usage flags**: Control WHERE items spawn (Military, Farm, Town, etc.) — mapped to building categories.
- **Value flags (Tiers)**: Control spawn zone restrictions. Tier1 = coast/starter areas, Tier4 = deep inland/high-value areas. Items without tier flags can spawn anywhere their usage allows.
- **Flags**: `count_in_cargo/hoarder/map/player` control how CE counts existing items. `deloot` marks dynamic-event-only loot.

## Scripting Language

Files use **Enforce Script** (`.c` files) — DayZ's C-like scripting language. Not standard C/C++. Class-based with `override`, `ref`, `autoptr`, and engine-specific APIs (`GetGame()`, `EntityAI`, `PlayerBase`, etc.).

## Project Goal

Replacing the vanilla Central Economy loot spawning system with a custom scripted solution. See `docs/CE-Replacement-Research.md` for full research, API reference, and implementation strategy.

## Key Script Files for CE Work

- **`scripts/3_game/ce/centraleconomy.c`** — CEApi class (spawn, lifetime, avoidance methods) + ECE/RF flag constants
- **`scripts/3_game/hive/hive.c`** — Hive class (persistence init, character save/load)
- **`scripts/4_world/entities/itembase.c`** — ItemBase with EEOnCECreate hook
- **`scripts/5_mission/mission/missionserver.c`** — MissionServer entry point
- **`scripts/config.cpp`** — Full game class hierarchy (64 KB)

## External Data

Full game data extraction at `C:\Users\Drifter\Documents\DayZ Projects\` — contains non-CE assets (models, textures, sounds, animations). All CE-relevant files are already in this repo.
