# DayZ Central Economy (CE) Loot System — Research & Replacement Strategy

## Table of Contents

- [How the CE Works](#how-the-ce-works)
  - [Overview](#overview)
  - [The Spawn Pipeline](#the-spawn-pipeline)
  - [CE Cycle Behavior](#ce-cycle-behavior)
- [Configuration Files Reference](#configuration-files-reference)
  - [Economy Control (`db/economy.xml`)](#economy-control-dbeconomyxml)
  - [Item Definitions (`db/types.xml`)](#item-definitions-dbtypesxml)
  - [Global Parameters (`db/globals.xml`)](#global-parameters-dbglobalsxml)
  - [Limits Definitions (`cfglimitsdefinition.xml`)](#limits-definitions-cfglimitsdefinitionxml)
  - [Spawnable Types (`cfgspawnabletypes.xml`)](#spawnable-types-cfgspawnabletypesxml)
  - [Random Presets (`cfgrandompresets.xml`)](#random-presets-cfgrandompresetsxml)
  - [Dynamic Events (`db/events.xml`)](#dynamic-events-dbeventsxml)
  - [Map Groups (`mapgroup*.xml`)](#map-groups-mapgroupxml)
  - [Area Flags (`areaflags.map`)](#area-flags-areaflagsmap)
- [Scripting API Reference](#scripting-api-reference)
  - [Hive](#hive)
  - [CEApi — Spawn Methods](#ceapi--spawn-methods)
  - [CEApi — Lifetime Management](#ceapi--lifetime-management)
  - [CEApi — Player/Vehicle Avoidance](#ceapi--playervehicle-avoidance)
  - [CEApi — Globals Access](#ceapi--globals-access)
  - [CEApi — Debug & Export](#ceapi--debug--export)
  - [CEItemProfile](#ceitemprofile)
  - [ECE Spawn Flags](#ece-spawn-flags)
  - [RF Rotation Flags](#rf-rotation-flags)
  - [EEOnCECreate Hook](#eeoncecreatehook)
  - [Timer Infrastructure](#timer-infrastructure)
- [Replacement Strategies](#replacement-strategies)
  - [Strategy 1: Full Replacement (Disable CE, Script Everything)](#strategy-1-full-replacement)
  - [Strategy 2: Hybrid (Keep Hive, Override Spawn Logic)](#strategy-2-hybrid-recommended)
  - [Strategy 3: XML-Level Override (Data Replacement Only)](#strategy-3-xml-level-override)
  - [Strategy 4: Direct Object Spawning (Bypass CE Entirely)](#strategy-4-direct-object-spawning)
- [Recommended Approach](#recommended-approach)
- [Existing Modding Precedent](#existing-modding-precedent)

---

## How the CE Works

### Overview

The Central Economy (CE) is a server-side native C++ engine that manages all item, vehicle, animal, zombie, and dynamic event spawning. It runs in periodic cycles (~5 minutes) and is entirely data-driven via XML configuration files.

The CE is initialized in `init.c` through the Hive:

```c
Hive ce = CreateHive();
if (ce)
    ce.InitOffline();
```

After initialization, the `CEApi` object is accessible via `GetCEApi()` and exposes all spawn, lifetime, and avoidance methods to EnfScript.

### The Spawn Pipeline

```
cfglimitsdefinition.xml     Defines valid categories (weapons, food...),
                            usage flags (Military, Town...), value tiers (Tier1-4),
                            and placement tags (floor, shelves, ground)
        │
        ▼
mapgroupproto.xml           Maps building model classes to loot spawn positions.
mapgrouppos.xml             Each position is tagged with categories + usages
mapgroupcluster*.xml        (e.g., "this shelf in a police station can spawn
                            'weapons' with 'Police' usage in Tier2")
        │
        ▼
db/types.xml                Each <type> entry declares: nominal count, min count,
                            lifetime, restock delay, cost (priority), category,
                            usage zones, and tier restrictions
        │
        ▼
CE Spawn Loop               Every ~5 min: for each type where current_count < nominal:
                              - Find valid spawn points matching category + usage + tier
                              - Exclude points near players (LootSpawnAvoidance radius)
                              - Spawn items respecting restock timer and cost priority
                              - Apply random damage between LootDamageMin and LootDamageMax
        │
        ▼
cfgspawnabletypes.xml       After spawning, attach random presets (scopes, mags, cargo)
cfgrandompresets.xml        from grouped preset pools
```

### CE Cycle Behavior

1. **Counting**: CE counts every tracked item in the world using the `count_in_*` flags from `types.xml`
2. **Cleanup**: Items whose lifetime has expired are removed
3. **Respawn**: For each item type below its `min` count, CE attempts to spawn new instances up toward `nominal`
4. **Proximity exclusion**: Spawn points within `LootSpawnAvoidance` (default 100m) of any player are skipped
5. **Restock timer**: An item type can only respawn after its `restock` seconds have elapsed since last spawn
6. **Cost priority**: Higher `cost` items get spawned first when multiple types need restocking

---

## Configuration Files Reference

### Economy Control (`db/economy.xml`)

Master on/off switches for each CE subsystem. Each category has four flags:

| Flag | Purpose |
|------|---------|
| `init` | Spawn items on server startup |
| `load` | Load persisted items from storage |
| `respawn` | Allow CE to respawn items during cycles |
| `save` | Persist items to storage on shutdown |

Current vanilla configuration:
```xml
<economy>
    <dynamic init="1" load="1" respawn="1" save="1"/>    <!-- loot items -->
    <animals init="1" load="0" respawn="1" save="0"/>
    <zombies init="1" load="0" respawn="1" save="0"/>
    <vehicles init="1" load="1" respawn="1" save="1"/>
    <randoms init="0" load="0" respawn="1" save="0"/>
    <custom init="0" load="0" respawn="0" save="0"/>
    <building init="1" load="1" respawn="0" save="1"/>
    <player init="1" load="1" respawn="1" save="1"/>
</economy>
```

**To disable CE loot spawning**, set `dynamic` to `init="0" respawn="0"` while keeping `load` and `save` for persistence.

### Item Definitions (`db/types.xml`)

The core loot table. Each `<type>` entry defines one spawnable item class:

```xml
<type name="AK101">
    <nominal>2</nominal>          <!-- target count on map -->
    <lifetime>28800</lifetime>    <!-- seconds before untouched item despawns (8 hours) -->
    <restock>3600</restock>       <!-- minimum seconds between respawn attempts (1 hour) -->
    <min>1</min>                  <!-- CE respawns when count drops below this -->
    <quantmin>30</quantmin>       <!-- min quantity % (-1 = ignore) -->
    <quantmax>80</quantmax>       <!-- max quantity % (-1 = ignore) -->
    <cost>100</cost>              <!-- priority/value (higher = spawned first) -->
    <flags count_in_cargo="0" count_in_hoarder="0" count_in_map="1"
           count_in_player="0" crafted="0" deloot="1"/>
    <category name="weapons"/>
    <usage name="Military"/>
    <value name="Tier3"/>
    <value name="Tier4"/>
</type>
```

**Key fields:**

| Field | Description |
|-------|-------------|
| `nominal` | Target item count on the map. CE spawns toward this number. |
| `min` | Minimum count. CE prioritizes respawning when count falls below this. |
| `lifetime` | Seconds before an untouched item on the ground is cleaned up. |
| `restock` | Minimum seconds between respawn attempts for this item type. |
| `quantmin`/`quantmax` | Quantity range as percentage (e.g., ammo in magazine). -1 = use default. |
| `cost` | Priority value. Higher cost items are spawned/preserved first. |
| `count_in_cargo` | Count items stored inside containers toward the total. |
| `count_in_hoarder` | Count items in player stashes/tents toward the total. |
| `count_in_map` | Count items on the ground toward the total. |
| `count_in_player` | Count items in player inventory toward the total. |
| `crafted` | Item is craftable (affects CE tracking). |
| `deloot` | Dynamic event loot only — won't spawn from regular CE cycle. |
| `category` | Item category — must match a category in `cfglimitsdefinition.xml`. |
| `usage` | Where the item spawns — mapped to building categories (Military, Town, etc.). Multiple allowed. |
| `value` | Tier restriction. Multiple allowed. Items without tiers can spawn anywhere usage allows. |

### Global Parameters (`db/globals.xml`)

Server-wide economy constants. Accessed in script via `GetCEApi().GetCEGlobalInt/Float/String()`.

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `AnimalMaxCount` | int | 200 | Maximum animals on map |
| `ZombieMaxCount` | int | 1000 | Maximum zombies on map |
| `CleanupAvoidance` | int | 100 | Don't clean up items within this radius (m) of players |
| `CleanupLifetimeDefault` | int | 45 | Default cleanup time (s) for dropped items |
| `CleanupLifetimeDeadPlayer` | int | 3600 | Cleanup time for dead player bodies |
| `CleanupLifetimeDeadInfected` | int | 330 | Cleanup time for dead zombie bodies |
| `CleanupLifetimeDeadAnimal` | int | 1200 | Cleanup time for dead animal bodies |
| `CleanupLifetimeRuined` | int | 330 | Cleanup time for ruined items |
| `CleanupLifetimeLimit` | int | 50 | Cleanup distance threshold |
| `InitialSpawn` | int | 100 | Percentage of nominals to spawn on first server start |
| `RestartSpawn` | int | 0 | Percentage of nominals to respawn on restart |
| `SpawnInitial` | int | 1200 | Initial spawn duration (s) |
| `LootSpawnAvoidance` | int | 100 | Don't spawn loot within this radius (m) of players |
| `LootProxyPlacement` | int | 1 | Enable proxy-based loot placement |
| `LootDamageMin` | float | 0.0 | Minimum random damage applied to spawned loot |
| `LootDamageMax` | float | 0.82 | Maximum random damage applied to spawned loot |
| `RespawnAttempt` | int | 2 | Max respawn attempts per CE cycle per type |
| `RespawnLimit` | int | 20 | Max total respawns per CE cycle |
| `RespawnTypes` | int | 12 | Max item types processed per CE cycle |
| `FoodDecay` | int | 1 | Enable food decay system |
| `WorldWetTempUpdate` | int | 1 | Enable wet/temp world updates |
| `FlagRefreshFrequency` | int | 432000 | Flag pole refresh interval (5 days) |
| `FlagRefreshMaxDuration` | int | 3456000 | Flag pole max lifetime (40 days) |
| `IdleModeCountdown` | int | 60 | Countdown before idle mode |
| `IdleModeStartup` | int | 1 | Enable idle mode on startup |
| `TimeLogin` | int | 15 | Login timeout (s) |
| `TimeLogout` | int | 15 | Logout timeout (s) |
| `TimePenalty` | int | 20 | Combat logout penalty (s) |
| `TimeHopping` | int | 60 | Server hopping penalty (s) |
| `ZoneSpawnDist` | int | 300 | Zombie spawn trigger distance (m) |

### Limits Definitions (`cfglimitsdefinition.xml`)

Defines the valid vocabulary for categories, tags, usages, and value tiers used across all other config files:

**Categories:** `tools`, `containers`, `clothes`, `lootdispatch`, `food`, `weapons`, `books`, `explosives`

**Tags:** `floor`, `shelves`, `ground`

**Usage flags (WHERE items spawn):**
`Military`, `Police`, `Medic`, `Firefighter`, `Industrial`, `Farm`, `Coast`, `Town`, `Village`, `Hunting`, `Office`, `School`, `Prison`, `Lunapark`, `SeasonalEvent`, `ContaminatedArea`, `Historical`

**Value flags (WHICH tier zone):**
`Tier1` (coast/starter), `Tier2`, `Tier3`, `Tier4` (deep inland/high-value), `Unique`

### Spawnable Types (`cfgspawnabletypes.xml`)

Controls what attachments and cargo spawn WITH an item. Applied after CE places the base item:

```xml
<type name="AKM">
    <attachments chance="0.3">
        <item name="AK_WoodBttstck" chance="0.5" />
        <item name="AK_PlasticBttstck" chance="0.3" />
    </attachments>
    <cargo chance="0.2">
        <item name="Mag_AKM_30Rnd" chance="0.4" />
    </cargo>
</type>
```

### Random Presets (`cfgrandompresets.xml`)

Defines grouped item pools that can be referenced by `cfgspawnabletypes.xml` for randomized cargo/attachment filling.

### Dynamic Events (`db/events.xml`)

Controls spawning of vehicles, helicopter crashes, animal herds, zombie hordes, and other dynamic events:

```xml
<event name="StaticHeliCrash">
    <nominal>3</nominal>          <!-- target count active on map -->
    <min>1</min>                  <!-- minimum before respawn -->
    <max>5</max>                  <!-- maximum allowed -->
    <lifetime>1500</lifetime>     <!-- seconds before cleanup -->
    <restock>300</restock>        <!-- seconds between respawn attempts -->
    <saferadius>500</saferadius>  <!-- minimum distance from players -->
    <distanceradius>500</distanceradius>  <!-- spacing between instances -->
    <cleanupradius>200</cleanupradius>
    <flags deletable="1" init_random="0" remove_damaged="1"/>
    <position>fixed</position>    <!-- fixed or player-relative -->
    <limit>mixed</limit>          <!-- child or mixed -->
    <active>1</active>            <!-- enabled -->
    <children>
        <child lootmax="10" lootmin="5" max="3" min="1" type="Wreck_Mi8"/>
    </children>
</event>
```

### Map Groups (`mapgroup*.xml`)

Binary/XML files mapping every building on the map to its loot spawn positions. Each spawn position has:
- A 3D coordinate relative to the building
- Category and usage tags determining what can spawn there
- A container type (floor, shelf, etc.)

These are typically generated by DayZ tools, not hand-edited.

### Area Flags (`areaflags.map`)

A binary bitmap defining the four tier zones across the Chernarus+ map. The CE reads this file to determine which tier a given map position falls into, controlling which `value` flags are active for spawn point selection.

See `lootTiers.jpg` in the repo root for a visual reference.

---

## Scripting API Reference

All methods sourced from `scripts/3_game/ce/centraleconomy.c` (769 lines) and `scripts/3_game/hive/hive.c` (31 lines).

### Hive

The Hive manages persistence, character data, and CE initialization.

```c
// Create and initialize
Hive ce = CreateHive();    // creates the Hive singleton
ce.InitOffline();          // offline/local mode (no database server)
ce.InitOnline(ceSetup, host);  // online mode with database
ce.InitSandbox();          // sandbox/test mode

// Character persistence
GetHive().CharacterSave(player);
GetHive().CharacterKill(player);
GetHive().CharacterExit(player);

// Configuration
GetHive().SetShardID(shard);
GetHive().SetEnviroment(env);

// State
GetHive().IsIdleMode();
GetHive().CharacterIsLoginPositionChanged(player);  // only valid during login

// Lifecycle
DestroyHive();
Hive hive = GetHive();     // get existing instance
```

### CEApi — Spawn Methods

All spawn methods are accessed through `GetCEApi()`. Most are marked DEVELOPER/DIAG ONLY in source but are functional.

```c
// Spawn loot item(s) — best for general item spawning
// count > 1 spawns in a circle of fRange radius around vPos
GetCEApi().SpawnLoot(className, pos, angle, count, range);

// Spawn entity — better for animals/infected/vehicles
GetCEApi().SpawnEntity(className, pos, range, count);

// Spawn single entity and get reference back
Object obj = GetCEApi().SpawnSingleEntity(className, pos);

// Spawn with rotation flags
GetCEApi().SpawnRotation(className, pos, range, count, rotationFlags);

// Spawn a dynamic event (heli crash, vehicle spawn, etc.)
// FORCE spawn — bypasses limit and avoidance checks
GetCEApi().SpawnDE(eventName, pos, angle);
GetCEApi().SpawnDEEx(eventName, pos, angle, eceFlags);

// Spawn a prototype group (building + its loot)
GetCEApi().SpawnGroup(groupName, pos, angle);

// Bulk debug spawns
GetCEApi().SpawnDynamic(pos, showCylinders, defaultDistance);  // all dynamic items
GetCEApi().SpawnVehicles(pos, showCylinders, defaultDistance); // all vehicles
GetCEApi().SpawnBuilding(pos, showCylinders, defaultDistance); // all buildings

// Performance test — spawns in grid from origin
GetCEApi().SpawnPerfTest(className, count);
```

### CEApi — Lifetime Management

```c
// Subtract seconds from lifetime of ALL items in world
GetCEApi().TimeShift(seconds);

// Override lifetime for any DE spawned after this call (0 = stop overriding)
GetCEApi().OverrideLifeTime(seconds);

// Modify lifetimes within a radius
GetCEApi().RadiusLifetimeIncrease(center, radius, seconds);
GetCEApi().RadiusLifetimeDecrease(center, radius, seconds);
GetCEApi().RadiusLifetimeReset(center, radius);  // reset to DB defaults

// Deplete lifetime of everything — queue full cleanup
GetCEApi().CleanMap();
```

Lifetime values are clamped to `[3, 316224000]` (3 seconds to 10 years).

### CEApi — Player/Vehicle Avoidance

```c
// Returns FALSE if a player is within distance (true = safe to spawn)
bool safe = GetCEApi().AvoidPlayer(pos, distance);

// Returns FALSE if a vehicle is within distance
// sDEName filters to specific DE; empty = all vehicles
bool safe = GetCEApi().AvoidVehicle(pos, distance, deName);

// Count players in range
int count = GetCEApi().CountPlayersWithinRange(pos, range);
```

### CEApi — Globals Access

Read values from `db/globals.xml` at runtime:

```c
int val    = GetCEApi().GetCEGlobalInt("AnimalMaxCount");      // type="0"
float val  = GetCEApi().GetCEGlobalFloat("LootDamageMax");     // type="1"
string val = GetCEApi().GetCEGlobalString("SomeStringVar");    // type="2"
```

### CEApi — Debug & Export

```c
// Log categories — outputs to storage/log/*.csv
GetCEApi().EconomyLog(EconomyLogCategories.Economy);
GetCEApi().EconomyLog(EconomyLogCategories.RespawnQueue);
GetCEApi().EconomyLog(EconomyLogCategories.Container);
// Full list: Economy, EconomyRespawn, RespawnQueue, Container, Matrix,
//   UniqueLoot, Bind, SetupFail, Storage, Classes, Category, Tag,
//   SCategory, STag, SAreaflags, SCrafted, MapGroup, MapComplete, InfectedZone

// Map visualization — outputs to storage/lmap/*.tga
GetCEApi().EconomyMap("Deagle");                              // specific item
GetCEApi().EconomyMap(EconomyMapStrings.ALL_LOOT);            // all loot
GetCEApi().EconomyMap(EconomyMapStrings.Category("food"));    // by category
GetCEApi().EconomyMap(EconomyMapStrings.Tag("floor"));        // by tag
// Full list: ALL_ALL, ALL_LOOT, ALL_VEHICLE, ALL_INFECTED, ALL_ANIMAL,
//   ALL_PLAYER, ALL_PROXY, ALL_PROXY_STATIC, ALL_PROXY_DYNAMIC, ALL_PROXY_ABANDONED

// Output debug info
GetCEApi().EconomyOutput(EconomyOutputStrings.STATUS, 0);     // overall CE stats
GetCEApi().EconomyOutput(EconomyOutputStrings.WORLD, 0);      // object counts by category
GetCEApi().EconomyOutput(EconomyOutputStrings.CLOSE, 0.3);    // loot points closer than radius
// Full list: LINKS, SUSPICIOUS, DE_CLOSE_POINT, ABANDONED, EMPTY, CLOSE, WORLD, STATUS, LOOT_SIZE

// Spawn analysis — generates images + logs for item spawn distribution
GetCEApi().SpawnAnalyze("FlatCap_Grey");  // specific item
GetCEApi().SpawnAnalyze("*");             // all items to CSV

// Export map data to storage/export/
GetCEApi().ExportProxyData(center, radius);   // mapgrouppos.xml
GetCEApi().ExportClusterData();               // mapgroupcluster.xml
GetCEApi().ExportProxyProto();                // mapgroupproto.xml
GetCEApi().ExportSpawnData();                 // spawnpoints.bin

// Proxy point management
GetCEApi().MarkCloseProxy(radius, allSelections);  // invalidate close points
GetCEApi().RemoveCloseProxy();                      // remove invalidated points
GetCEApi().ListCloseProxy(radius);                  // list close points to log
```

### CEItemProfile

Per-item economy data from `types.xml`, accessed through an entity's economy profile:

```c
CEItemProfile profile = entityAI.GetEconomyProfile();
if (profile)
{
    int nominal   = profile.GetNominal();
    int min       = profile.GetMin();
    float qtyMin  = profile.GetQuantityMin();   // 0.0-1.0
    float qtyMax  = profile.GetQuantityMax();   // 0.0-1.0
    float qty     = profile.GetQuantity();       // random between min/max
    float lifetime = profile.GetLifetime();      // seconds
    float restock  = profile.GetRestock();       // seconds
    int cost       = profile.GetCost();
    int usageFlags = profile.GetUsageFlags();    // bitmask
    int valueFlags = profile.GetValueFlags();    // bitmask
}
```

### ECE Spawn Flags

Flags used with `GetGame().CreateObjectEx()` and CE spawn methods:

```c
// Individual flags
ECE_NONE                  = 0
ECE_SETUP                 = 2        // full entity setup (use for NEW entities)
ECE_TRACE                 = 4        // trace under entity for surface placement
ECE_CENTER                = 8        // use model center for placement
ECE_UPDATEPATHGRAPH       = 32       // update navmesh
ECE_ROTATIONFLAGS         = 512      // enable rotation flags
ECE_CREATEPHYSICS         = 1024     // create collision/physics
ECE_INITAI                = 2048     // initialize AI
ECE_AIRBORNE              = 4096     // airborne placement
ECE_EQUIP_ATTACHMENTS     = 8192     // equip configured attachments
ECE_EQUIP_CARGO           = 16384    // equip configured cargo
ECE_EQUIP                 = 24576    // attachments + cargo
ECE_EQUIP_CONTAINER       = 2097152  // populate DE container during spawn
ECE_NOSURFACEALIGN        = 262144   // don't align to surface
ECE_KEEPHEIGHT            = 524288   // keep creation height
ECE_NOLIFETIME            = 4194304  // don't set lifetime
ECE_NOPERSISTENCY_WORLD   = 8388608  // don't save in world
ECE_NOPERSISTENCY_CHAR    = 16777216 // don't save in character
ECE_DYNAMIC_PERSISTENCY   = 33554432 // no persistence until player takes it
ECE_LOCAL                 = 1073741824 // local-only entity

// Predefined combinations
ECE_IN_INVENTORY          = 787456   // CREATEPHYSICS|KEEPHEIGHT|NOSURFACEALIGN
ECE_PLACE_ON_SURFACE      = 1060     // CREATEPHYSICS|UPDATEPATHGRAPH|TRACE
ECE_OBJECT_SWAP           = 787488   // CREATEPHYSICS|UPDATEPATHGRAPH|KEEPHEIGHT|NOSURFACEALIGN
ECE_FULL                  = 25126    // SETUP|TRACE|ROTATIONFLAGS|UPDATEPATHGRAPH|EQUIP
```

### RF Rotation Flags

```c
RF_NONE        = 0
RF_FRONT       = 1       // front side placement
RF_TOP         = 2       // top side placement
RF_LEFT        = 4       // left side placement
RF_RIGHT       = 8       // right side placement
RF_BACK        = 16      // back side placement
RF_BOTTOM      = 32      // bottom side placement
RF_ALL         = 63      // all sides
RF_IGNORE      = 64      // ignore RF flags, spawn as modeled
RF_RANDOMROT   = 64      // random rotation around axis
RF_ORIGINAL    = 128     // use default object placement
RF_DECORRECTION = 256    // angle correction for building-placed items
RF_DEFAULT     = 512     // use default config placement

// Combinations
RF_TOPBOTTOM   = 34      // TOP|BOTTOM
RF_LEFTRIGHT   = 12      // LEFT|RIGHT
RF_FRONTBACK   = 17      // FRONT|BACK
```

### EEOnCECreate Hook

Called on an entity when the CE spawns it. Override in item subclasses for spawn-time customization. 57 files in the game scripts override this method.

**Base implementation** (`scripts/4_world/entities/itembase.c`):
```c
override void EEOnCECreate()
{
    if (!IsMagazine())
        SetCEBasedQuantity();   // sets quantity from CE profile
    SetZoneDamageCEInit();      // applies zone-based damage
}
```

**Other notable overrides:**
- `CarScript` / `BoatScript` — set random fuel levels
- `MushroomBase` — set spoilage state
- Edible items (Apple, Pear, Tomato, etc.) — set contamination state
- `Canteen` / `WaterBottle` — set water fill levels
- `ContaminatedArea_Dynamic` — set up PPE effects
- `FireworksBase` — set orientation
- `CrashBase` (heli crashes) — no custom logic but hook is present
- `Wreck_SantasSleigh` — seasonal event setup

### Timer Infrastructure

From `scripts/2_gamelib/tools.c` — used to build recurring spawn loops:

```c
// Deferred call (one-shot or repeating)
GetGame().GetCallQueue(CALL_CATEGORY_GAMEPLAY).CallLater(
    function,    // method to call
    delayMs,     // delay in milliseconds
    repeat,      // true = repeat, false = one-shot
    param1...9   // optional parameters
);

// Timer class
ref Timer m_Timer = new Timer();
m_Timer.Run(
    durationSeconds,   // interval
    thisObject,        // object owning the callback
    "MethodName",      // callback method name (string)
    params,            // Param object or null
    loop               // true = repeat
);
m_Timer.Pause();
m_Timer.Continue();
m_Timer.Stop();
bool running = m_Timer.IsRunning();
```

---

## Replacement Strategies

### Strategy 1: Full Replacement

Disable the CE entirely and script all spawning from scratch.

**How:**
1. Set all flags to `0` in `economy.xml`:
   ```xml
   <dynamic init="0" load="0" respawn="0" save="0"/>
   ```
2. Build a `modded class MissionServer` with a timer-based spawn loop
3. Maintain your own item tracking, spawn tables, and cleanup logic

**Pros:** Total control over every aspect of spawning — algorithm, timing, distribution, rarity curves.

**Cons:** Must reimplement persistence, cleanup, network sync, and item counting. High complexity, high risk of bugs.

### Strategy 2: Hybrid (Recommended)

Keep the Hive infrastructure for persistence and cleanup, but disable CE's automatic spawning and replace it with custom logic.

**How:**
1. Keep `CreateHive()` / `InitOffline()` in `init.c`
2. Disable CE spawning in `economy.xml`:
   ```xml
   <dynamic init="0" load="1" respawn="0" save="1"/>
   ```
   (keep `load` and `save` for persistence across restarts)
3. Build a `modded class MissionServer` with a timer-based spawn loop:
   ```c
   modded class MissionServer
   {
       ref Timer m_SpawnTimer;

       override void OnInit()
       {
           super.OnInit();
           m_SpawnTimer = new Timer();
           m_SpawnTimer.Run(300, this, "RunCustomLootCycle", null, true);
       }

       void RunCustomLootCycle()
       {
           // For each item in custom loot table:
           //   Check GetCEApi().AvoidPlayer(pos, 100)
           //   Spawn via GetCEApi().SpawnLoot() or CreateObjectEx()
           //   Set lifetime via entity.SetLifetime()
       }
   }
   ```
4. Use `GetCEApi()` methods for spawning, proximity checks, and lifetime management
5. Define custom loot tables in script or custom XML format
6. Override `EEOnCECreate()` on item classes for spawn-time initialization

**Pros:** Full algorithmic control while leveraging DayZ's built-in persistence, cleanup, and network sync. Moderate complexity.

**Cons:** Still somewhat coupled to CE infrastructure. Must understand which CE features remain active.

### Strategy 3: XML-Level Override

Replace only the data files, keeping the CE algorithm intact. Since DayZ 1.08+, custom `<ce>` folder entries in `cfgeconomycore.xml` allow modular file replacement:

```xml
<ce folder="custom">
    <file name="my_types.xml" type="types" />
    <file name="my_events.xml" type="events" />
</ce>
```

**Pros:** Clean mod separation, easy to distribute, no script changes needed.

**Cons:** Can only change WHAT spawns and WHERE — cannot change HOW the CE algorithm works. No custom rarity curves, dynamic difficulty, or player-responsive logic.

### Strategy 4: Direct Object Spawning

Skip the CE completely. Don't call `CreateHive()`. Use raw `CreateObjectEx()`:

```c
void SpawnItem(string className, vector pos)
{
    EntityAI item = EntityAI.Cast(
        GetGame().CreateObjectEx(className, pos, ECE_PLACE_ON_SURFACE)
    );
    if (item)
        item.SetLifetime(3600);
}
```

**Pros:** Zero CE dependency. Complete freedom.

**Cons:** No persistence across restarts. No automatic cleanup. No economy tracking. Must build everything from scratch — essentially writing a complete game economy engine.

---

## Recommended Approach

**Strategy 2 (Hybrid)** is the most practical path for a custom loot system:

| Concern | Solution |
|---------|----------|
| Persistence | Handled by Hive (`load="1" save="1"`) |
| Cleanup/Lifetime | Handled by CE engine (still runs even with `respawn="0"`) |
| Item Spawning | Custom script using `GetCEApi().SpawnLoot()` / `CreateObjectEx()` |
| Player Avoidance | `GetCEApi().AvoidPlayer()` / `CountPlayersWithinRange()` |
| Spawn Timing | `Timer` class in `modded class MissionServer` |
| Loot Tables | Custom format (script arrays, custom XML, or JSON) |
| Spawn-time Setup | `EEOnCECreate()` overrides on item classes |
| Types.xml Data | Still readable via `EntityAI.GetEconomyProfile()` if needed |

This gives full algorithmic control while keeping DayZ's infrastructure for the hard parts (persistence, network sync, cleanup).

---

## Existing Modding Precedent

No widely-known mod completely replaces the CE loot system. The closest examples:

- **PvZmoD_Spawn_System** — Custom zombie spawning system, uses CEApi but focuses on infected rather than loot
- **DayZ Expansion** — Adds parallel economy layers (traders, missions, airdrops) alongside CE, not as a replacement
- **TraderPlus** — Adds trader-based economy alongside CE
- **Community Framework** — Provides `MissionServer` extensions and utility classes useful for custom systems

All of these work alongside the CE rather than replacing it, which validates the hybrid approach — the CE's persistence and cleanup infrastructure is valuable enough that even major mods keep it running.
