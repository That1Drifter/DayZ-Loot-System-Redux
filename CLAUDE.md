# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

DayZ-Loot-System-Redux is a server-side loot economy configuration for DayZ Standalone (Chernarus+ map). It defines item spawn rates, loot distribution tiers, animal/zombie territories, and player spawn equipment. There is no traditional build/test/lint pipeline — this is entirely XML configuration plus one EnfScript (C-like) initialization file.

## Build

The project uses an Ant build file (`dayzOffline.chernarusplus/build.xml`) that packages into a `.pbo` addon. The build depends on an external DayZ data pipeline (`${src.root.dir}/build/buildData.xml`) not included in this repo. To deploy, copy the `dayzOffline.chernarusplus/` folder into a DayZ server's mission directory.

## Architecture

All files live under `dayzOffline.chernarusplus/`:

- **`init.c`** — Mission entry point (EnfScript). Initializes weather, economy Hive, date reset, and defines `CustomMission` class with player character creation and starting equipment (rags, random chemlight, random fruit).

- **`db/types.xml`** (~503KB) — The core loot table. Each `<type>` defines an item's nominal count, lifetime, restock timer, min count, cost, flags, category, usage zone, and value tier. This is the primary file for loot balance changes.

- **`db/events.xml`** — Zombie, animal, and vehicle spawn events with quantities, timing, and zone references.

- **`db/globals.xml`** — Server-wide variables: zombie/animal max counts, cleanup lifetimes, respawn limits, login/logout timers.

- **`db/economy.xml`** — Toggles for economy subsystems (dynamic loot, animals, zombies, vehicles, player persistence).

- **`cfgspawnabletypes.xml`** — Maps items to container types (barrels, tents, bags) with attachment/cargo presets.

- **`cfgrandompresets.xml`** — Loot presets grouped by location type (village, city, army, police, hunter, medic, hermit, industrial).

- **`cfgeventspawns.xml`** — Hardcoded world coordinates for event spawn positions.

- **`cfgenvironment.xml`** + **`env/*.xml`** — Animal and zombie territory definitions with spatial boundaries.

- **`mapgroupcluster*.xml`**, **`mapgrouppos.xml`**, **`mapgroupproto.xml`**, **`mapclusterproto.xml`** — Map zone data defining building groups and loot spawn positions. These are large generated files; avoid manual editing.

## Loot Tier System

Items are assigned to tiers (Tier1–Tier4) via the `<value>` tag in `db/types.xml`. The `<usage>` tag defines where items spawn (Military, Police, Hunting, Village, Town, etc.). See `lootTiers.jpg` for a visual map of tier zones across Chernarus+.

## Key Conventions

- Item entries in `types.xml` with only `<lifetime>` and `<flags crafted="1">` (no `<nominal>`) are craft-only items that don't spawn naturally.
- `quantmin`/`quantmax` of `-1` means the item spawns with default quantity.
- `count_in_map="1"` is the standard flag for tracking item counts in the economy; `count_in_hoarder`/`count_in_cargo`/`count_in_player` control whether stashed/carried items count toward limits.
