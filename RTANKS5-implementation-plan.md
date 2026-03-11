# R-TANKS 5: Open World Edition

## TINS Implementation Plan

> **There Is No Source** -- This document is the single source of truth for transforming R-TANKS 4 (arena combat) into R-TANKS 5 (open world tank RPG). Each step is self-contained and designed to be executed sequentially by an AI code agent with access to the existing codebase and the JSX component subfolders listed below.

---

## Description

R-TANKS 5 transforms the enclosed arena tank combat game into a boundless procedurally-generated open world. The player starts at coordinates `(0, 0)` in their **home base** (built and customised via the Level Editor) and drives out into an infinite terrain of valleys, mountains, and seeded dungeons filled with loot, NPCs, enemies, and secrets. All world content is generated deterministically from position using the ZeroBytes "position-as-seed" paradigm -- zero storage, infinite world.

The existing 5-weapon system, tank physics, boost/drift mechanics, and zustand state management are retained. The 10 hardcoded arenas are removed. Combat at scale is handled by an instanced particle system that resolves battles from seed alone. A wingman air-support ability, SNES-style SFX, procedural engine audio, RPG dialog, neon laser-frame UI, an attitude indicator HUD, and cinematic explosions complete the experience.

---

## Tech Stack (Unchanged)

| Layer | Technology |
|-------|-----------|
| UI Framework | React 18 |
| Language | TypeScript |
| 3D Rendering | Three.js + @react-three/fiber + drei |
| State Management | Zustand |
| Build Tool | Vite |

**No new npm dependencies are required.** All JSX components are vendored into `src/` and adapted to TypeScript.

---

## JSX Component Reference Folders

Each folder below is cloned into the project root. It contains a `.jsx` source file, a `-integration.md` guide, a `README.md`, and a `demo.html`. The integration guide for each component should be read at the start of its dedicated step. **Delete each folder after its step is complete.**

| Folder | Purpose in R-TANKS 5 | Key File |
|--------|----------------------|----------|
| `3D-attitude-indicator-JSX/` | HUD orientation compass | `AttitudeIndicatorCanvas.jsx` |
| `3D-Procedural-Terrain-V2-JSX/` | Infinite terrain generation | `ProceduralTerrainV2.jsx` |
| `EngineSounds-JSX/` | Tank engine audio | `audio-engine.jsx` |
| `ExplosionEngine-JSX/` | Enemy/destructible explosions | `ExplosionEngine.jsx` |
| `LaserFrame-JSX/` | Neon UI frame drawing effect | `laserprint-demo.jsx` |
| `PuppetJSX-2-zerobytes/` | World seed: valleys, rooms, loot, spawns | `PuppetJSX_Zerobytes.jsx` |
| `RPGdialog-JSX/` | NPC dialog system (2D portraits) | `RPGDialogue.jsx` |
| `SNESaudio-JSX/` | 16-bit synthesised SFX | `SNESSoundGenerator.jsx` |
| `WingmanSupport-JSX/` | Air support ability (4 wingmen) | `WingmanSupport.jsx` |
| `zerobytes-3dcombatlayer/` | Large-scale seeded combat + instanced VFX | `ZB-3DCombatLayerV3.jsx` |

---

## Architecture Overview

```
src/
  game/
    GameState.ts          -- Zustand store (extended for open world)
    Physics.ts            -- Tank movement (updated for terrain height sampling)
    WeaponSystem.ts       -- Retained: 5 weapons, hitscan, projectile pool
    AISystem.ts           -- Updated: enemies from seed, patrol valleys
    Input.ts              -- Retained + new keybind: wingman call (Q)
    SecretSystem.ts       -- Updated: secrets from seed per valley
  world/
    WorldSeed.ts          -- ZeroBytes core: coordsToSeed, deriveValues, hashFloat, coherentValue
    ValleyGenerator.ts    -- PuppetJSX room system scaled to valleys (mountains = walls)
    TerrainSystem.ts      -- Procedural terrain chunk manager with LOD
    HomeBase.ts           -- Level editor arena at (0,0) saved to localStorage
    LootSystem.ts         -- Seeded loot tables, items, pickups
    NPCSystem.ts          -- Seeded NPC spawns with dialog trees
  combat/
    CombatLayer.ts        -- ZeroBytes 3D combat: parallel resolution, instanced VFX
    WingmanSystem.ts      -- 4 wingmen, cooldown, fly-in attack, fly-away
    ExplosionManager.ts   -- Pooled explosions with wreckage
  audio/
    SNESAudio.ts          -- SNES synthesis engine + presets
    EngineAudio.ts        -- Tank engine sound (throttle, gear, idle, boost)
  components/
    3d/
      Terrain.tsx         -- React Three Fiber terrain mesh with chunk streaming
      Tank.tsx            -- Existing tank mesh (retained)
      EnemyTank.tsx       -- Instanced enemy tank rendering
      Skybox.tsx          -- Retained
      ValleyProps.tsx     -- Valley decorations: covers, landmarks, structures
    ui/
      LaserFrameUI.tsx    -- SVG laser-print frame effect for all UI panels
      AttitudeHUD.tsx     -- Canvas-based orientation indicator
      DialogOverlay.tsx   -- RPG dialog toast/fullscreen with portraits
      WorldMapHUD.tsx     -- Minimap showing nearby valleys
      WingmanHUD.tsx      -- Wingman selector + cooldown bar
      Button.tsx          -- Retained
      HUD.tsx             -- Updated with new elements
  screens/
    MainMenuScreen.tsx    -- Updated title: "R-TANKS 5"
    HomeBaseScreen.tsx    -- Replaces TheBunkerScreen: loadout + level editor
    OpenWorldScreen.tsx   -- Replaces BattleTanksScreen: main game loop
    ResultsScreen.tsx     -- Updated for open world stats
    SettingsScreen.tsx    -- Retained + new audio sliders
  data/
    arenas.ts             -- DELETED (arenas replaced by procedural world)
    npcDialogs.ts         -- Dialog trees for NPC types
    itemDatabase.ts       -- Loot item definitions
  utils/
    types.ts              -- Extended with open world types
    constants.ts          -- Extended with world constants
```

---

## Functionality

### Core Game Loop (OpenWorldScreen)

Each frame in `useFrame`:
1. **Input sampling** -- late-sample keyboard + mouse via `InputManager`
2. **Terrain height query** -- sample terrain height at player position for gravity/grounding
3. **Physics update** -- move player tank with terrain-following, boost, drift, wall-slide against mountains
4. **AI update** (10 Hz) -- enemies in loaded valleys patrol, chase, attack
5. **Weapon update** -- hitscan fire, projectile pool tick, damage application
6. **Combat layer update** -- resolve large-scale seeded battles in view frustum
7. **Wingman update** -- if active, animate wingman fly-in, attack, fly-away
8. **Secret/loot check** -- proximity test for discoverable items
9. **Chunk streaming** -- load/unload terrain chunks based on player position
10. **Audio update** -- engine sound parameters from throttle/speed, SFX triggers
11. **HUD update** -- attitude indicator, minimap, health/shield, ammo heat

### World Generation

The world is an infinite grid of **valleys**. Each valley is 500x500 world units (roughly 3x the old arena size). A valley's content is fully determined by its grid coordinates `(valleyX, valleyZ)` plus the master `worldSeed`.

**Valley = Room from PuppetJSX scaled up:**
- PuppetJSX `generateRoom(x, y, floor, seed)` becomes `generateValley(valleyX, valleyZ, worldSeed)`
- Room `width/height` (tiles) become valley dimensions (always 500x500 units)
- Room `type` maps to valley biome/purpose (combat, treasure, settlement, boss, etc.)
- Room `connections` (N/S/E/W doors) become mountain passes between valleys
- Room `enemyCount` scales to enemy tank spawns
- Room `treasureValue` determines loot density
- Room tiles become terrain features (floor=flat, wall=mountain ridge, water=lake, lava=hazard zone)

**Home Base (0,0):**
- The valley at grid `(0, 0)` is the player's home base
- Its terrain is flat (no procedural noise) and uses the **Level Editor** for customisation
- Player spawn is always at `(0, 0, 0)` world position
- Structures placed in editor are saved to `localStorage` key `rt-home-base`
- Mountains still ring the perimeter, with passes to adjacent valleys

### Terrain System

Adapted from `ProceduralTerrainV2.jsx`:
- **Chunks**: The world is divided into 50x50 unit chunks. Only chunks within render distance (400 units) are active.
- **Noise**: `SeededRNG` + fractal noise from the terrain component, biome-tinted per valley
- **Height sampling**: `getTerrainHeight(worldX, worldZ)` used by Physics.ts for tank grounding
- **Mountains as walls**: Valley borders are steep ridges (height 30+). Passes exist where `connections` are true.
- **Biomes**: Each valley has a biome determined by seed: grassland, desert, tundra, volcanic, alien, canyon (the 6 biomes from ProceduralTerrainV2)
- **Structures on terrain**: Terrain masking flattens ground under structures (same system as ProceduralTerrainV2 grid masking)
- **LOD**: Chunks far from player use resolution 32; near chunks use 64; immediate chunks use 96

### Combat System

Two tiers of combat coexist:

**Tier 1 -- Direct Combat (retained from R-TANKS 4):**
- Player's 5 weapons (Precision Cannon, Plasma Mortar, Rapid Pulse, Scatter Shot, Seeker Missile)
- Hitscan with range falloff, cooldowns, overheat
- AI enemies with 5 states (patrol, aggressive_chase, strafe_attack, flee, seek_cover)
- Enemies in the player's **current valley** and adjacent valleys are fully simulated

**Tier 2 -- Seeded Mass Combat (from zerobytes-3dcombatlayer):**
- Enemy forces beyond direct combat range are resolved via position-as-seed
- `resolveAttack3D(attacker, target, attackId, worldSeed)` determines outcomes deterministically
- Visual feedback: instanced particle effects (explosions, tracers) at near-zero GPU cost
- Player sees distant battles as instanced flash/smoke particles
- When player enters a valley with active seed-combat, results are pre-computed: some enemies already defeated, structures damaged

### Wingman Air Support

Adapted from `WingmanSupport.jsx`:
- **4 wingmen** selectable via number keys `6`, `7`, `8`, `9` (weapons are `1-5`)
- **Trigger**: Press `Q` to call selected wingman
- **Cooldown**: 45 seconds between calls
- **Behaviour**: Wingman ship flies in from behind player, performs attack run on enemies ahead, flies away
- **Attack types** (one per wingman):
  1. **Gatling** -- rapid projectiles in a cone ahead
  2. **Beam** -- sustained energy beam sweep
  3. **Lightning Chain** -- chains between nearby enemies
  4. **Missile Barrage** -- homing missiles at multiple targets
- **Integration**: `WingmanSystem.ts` manages state; `WingmanShip` is a 3D component in the scene

### Audio

**Engine Audio** (from `EngineSounds-JSX/audio-engine.jsx`):
- Web Audio API oscillator-based synthesis, no samples
- Maps tank state to audio: `throttle` (0-1 from W key), `speed` (normalized), `brake` (S key), `turning` (A/D), `gear` (auto from speed)
- Boost triggers gear-shift sound + higher rev
- Idle hum when stationary

**SFX** (from `SNESaudio-JSX/SNESSoundGenerator.jsx`):
- `SNESAudioEngine` class with `playPreset(preset)` method
- Map game events to presets:

| Game Event | SNES Preset |
|-----------|-------------|
| Weapon fire (Precision) | `laser` |
| Weapon fire (Plasma Mortar) | `explosion` |
| Weapon fire (Rapid Pulse) | `blaster` |
| Weapon fire (Scatter Shot) | `punch` |
| Weapon fire (Seeker Missile) | `magic` |
| Enemy hit | `enemyHit` |
| Enemy destroyed | `boom` |
| Player damaged | `hurt` |
| Player death | `death` |
| Secret found | `chest` |
| Loot pickup | `coin` |
| Level up / Power up | `powerUp` |
| Menu select | `select` |
| Menu confirm | `confirm` |
| Menu cancel | `cancel` |
| Boost activate | `dash` |
| Shield recharge | `heal` |
| NPC dialog open | `text` |
| Wingman called | `powerUp` |
| Valley enter | `teleport` |

### UI System

All UI panels use the **LaserFrame** effect from `LaserFrame-JSX`:
- SVG-based laser beam + hologram panel construction
- `LaserBeam` component fires from origin to panel corners
- `HologramPanel` orchestrates beams then reveals content with CRT scanline overlay
- Used for: main menu, settings, loadout, dialog boxes, results screen, inventory popups
- TRON neon palette: cyan `#00FFFF`, magenta `#FF00FF`, orange `#FF8C00`

**Attitude Indicator HUD** (from `3D-attitude-indicator-JSX`):
- Canvas 2D rendering (not Three.js) for performance
- Shows pitch, roll, heading of tank
- Repurposed fields: `airspeed` = tank speed in units/s, `altitude` = terrain height, `waypoint` = current valley name, `distance` = distance to home base
- Size: 150px in bottom-right corner of screen

**RPG Dialog** (from `RPGdialog-JSX`):
- Toast mode for in-game NPC encounters (bottom-left)
- Fullscreen mode for story moments
- Typewriter text, animated portrait frames
- Dialog trees with branching choices (merchant: buy/sell, quest-giver: accept/decline)
- NPC types seeded per valley: merchant, mechanic, scout, quest-giver

### Items & Loot

Seeded from PuppetJSX loot generation (`deriveValues` with salt `200000`):

| Item Type | Effect | Rarity Distribution |
|-----------|--------|---------------------|
| Health Kit | Restore 25/50/100 HP | Common 40%, Uncommon 30%, Rare 20%, Epic 10% |
| Shield Cell | Restore 25/50 shield | Common 50%, Uncommon 35%, Rare 15% |
| Damage Amp | 1.5x/2x/3x damage for 15s | Rare 60%, Epic 30%, Legendary 10% |
| Speed Boost | 1.3x/1.5x speed for 20s | Uncommon 50%, Rare 40%, Epic 10% |
| Armor Plate | +10/+20 armor permanent | Rare 70%, Epic 25%, Legendary 5% |
| Wingman Charge | Instant wingman cooldown reset | Epic 80%, Legendary 20% |

Items appear as glowing pickup meshes (icosahedron geometry, pulsing emissive, rarity-tinted color).

---

## Implementation Steps

Each step is designed to be completed in a single session. Steps are ordered by dependency. **Read the referenced integration guide at the start of each step.**

---

### Step 0: Project Scaffolding & Type Foundation

**Goal**: Create new directory structure, extend types, update constants. No functionality changes yet.

**Actions**:

1. Create directories:
   ```
   src/world/
   src/combat/
   src/audio/
   src/data/
   ```

2. **Extend `src/utils/types.ts`** -- add these new types:

```typescript
// --- Open World Types ---

export type ValleyBiome = 'grassland' | 'desert' | 'tundra' | 'volcanic' | 'alien' | 'canyon';

export type ValleyType = 'combat' | 'treasure' | 'settlement' | 'boss' | 'empty' | 'hazard';

export type ItemRarity = 'common' | 'uncommon' | 'rare' | 'epic' | 'legendary';

export type ItemType = 'health_kit' | 'shield_cell' | 'damage_amp' | 'speed_boost' | 'armor_plate' | 'wingman_charge';

export type NPCType = 'merchant' | 'mechanic' | 'scout' | 'quest_giver';

export type WingmanType = 'gatling' | 'beam' | 'lightning' | 'missile';

export interface ValleyDef {
  id: string;
  gridX: number;
  gridZ: number;
  seed: number;
  type: ValleyType;
  biome: ValleyBiome;
  difficulty: number;
  enemyCount: number;
  treasureValue: number;
  connections: { north: boolean; south: boolean; east: boolean; west: boolean };
  npcSpawns: NPCSpawn[];
  lootSpawns: LootSpawn[];
  enemySpawns: Vec3[];
  landmarks: LandmarkDef[];
  secrets: Secret[];
}

export interface NPCSpawn {
  id: string;
  type: NPCType;
  position: Vec3;
  dialogTreeId: string;
  name: string;
}

export interface LootSpawn {
  id: string;
  position: Vec3;
  item: ItemDef;
  collected: boolean;
}

export interface ItemDef {
  id: string;
  type: ItemType;
  name: string;
  rarity: ItemRarity;
  value: number;
  effect: { stat: string; amount: number; duration?: number };
}

export interface TerrainChunk {
  id: string;
  worldX: number;
  worldZ: number;
  resolution: number;
  heightData: Float32Array;
  biome: ValleyBiome;
  dirty: boolean;
}

export interface WingmanDef {
  id: number;
  name: string;
  type: WingmanType;
  color: string;
  attackDescription: string;
}

export interface DialogNode {
  id: string;
  character: string;
  text: string;
  portraitFrames?: string[];
  choices?: { label: string; nextNodeId: string }[];
  action?: string;
}

export interface DialogTree {
  id: string;
  nodes: Record<string, DialogNode>;
  startNodeId: string;
}

export interface PlayerProgress {
  worldSeed: number;
  position: Vec3;
  visitedValleys: string[];
  collectedLoot: string[];
  discoveredSecrets: string[];
  completedQuests: string[];
  inventory: ItemDef[];
  totalKills: number;
  playtime: number;
}
```

3. **Extend `src/utils/constants.ts`** -- add:

```typescript
// --- Open World ---
export const WORLD_SEED = 54321;
export const VALLEY_SIZE = 500;
export const VALLEY_MOUNTAIN_HEIGHT = 35;
export const VALLEY_PASS_WIDTH = 40;
export const CHUNK_SIZE = 50;
export const RENDER_DISTANCE = 400;
export const TERRAIN_LOD_NEAR = 96;
export const TERRAIN_LOD_MID = 64;
export const TERRAIN_LOD_FAR = 32;
export const LOD_NEAR_DIST = 100;
export const LOD_MID_DIST = 250;

// --- Wingmen ---
export const WINGMAN_COOLDOWN = 45000;
export const WINGMAN_ATTACK_DURATION = 3000;
export const WINGMEN: WingmanDef[] = [
  { id: 0, name: 'HAWK', type: 'gatling', color: '#FF6600', attackDescription: 'Rapid gatling burst' },
  { id: 1, name: 'VIPER', type: 'beam', color: '#00FFFF', attackDescription: 'Sustained energy beam' },
  { id: 2, name: 'STORM', type: 'lightning', color: '#FF00FF', attackDescription: 'Chain lightning' },
  { id: 3, name: 'NOVA', type: 'missile', color: '#00FF00', attackDescription: 'Homing missile barrage' },
];

// --- Audio ---
export const ENGINE_IDLE_RPM = 800;
export const ENGINE_MAX_RPM = 6000;
export const ENGINE_GEARS = 5;

// --- Items ---
export const MAX_INVENTORY = 20;
export const LOOT_PICKUP_RADIUS = 5;
export const NPC_INTERACT_RADIUS = 8;
```

4. **Update `ScreenName` type** in `types.ts`:
```typescript
export type ScreenName =
  | 'main_menu'
  | 'home_base'      // replaces 'the_bunker'
  | 'open_world'     // replaces 'battle_tanks'
  | 'results'
  | 'level_editor'   // now edits home base only
  | 'settings';
```

5. Delete `src/data/arenas.ts` (the 10 hardcoded arenas).

6. Update screen references in `GameState.ts` and `App.tsx` to use the new screen names.

---

### Step 1: World Seed System (PuppetJSX-2-zerobytes)

**Reference**: Read `PuppetJSX-2-zerobytes/PuppetJSX-2-zerobytes-integration.md`

**Goal**: Create `src/world/WorldSeed.ts` -- the deterministic generation backbone.

**Actions**:

1. Extract these pure functions from `PuppetJSX_Zerobytes.jsx` and convert to TypeScript in `src/world/WorldSeed.ts`:
   - `coordsToSeed(x, y, z, salt)` -- hash coordinates to integer seed
   - `mulberry32(seed)` -- seeded PRNG returning next-function
   - `deriveValues(x, y, z, salt, count)` -- get N random floats from position
   - `hashFloat(x, y, z, salt)` -- single random float [0,1) from position
   - `coherentValue(x, y, seed)` -- smooth noise for regional properties

2. Create `src/world/ValleyGenerator.ts`:
   - `generateValley(valleyX: number, valleyZ: number, worldSeed: number): ValleyDef`
   - Adapts PuppetJSX's `generateRoom()` logic:
     - Uses `coordsToSeed(valleyX, valleyZ, 0, worldSeed)` as valley seed
     - `deriveValues(valleyX, valleyZ, 0, worldSeed, 20)` for all valley properties
     - Room type mapping: `combat` (weight 35), `treasure` (15), `settlement` (10), `boss` (5), `empty` (25), `hazard` (10)
     - Connections: `hashFloat(valleyX, valleyZ, 0, worldSeed + direction_salt) > 0.3` for each cardinal direction
     - Valley at `(0, 0)` always returns `type: 'settlement'` with all connections true (home base)
     - Biome: `coherentValue(valleyX * 0.15, valleyZ * 0.15, worldSeed)` maps to 6 biome ranges
     - Enemy count: `3 + Math.floor(rng[5] * 12)` for combat valleys, 0 for settlements
     - Treasure value: `Math.floor(rng[6] * 100)` scaled by difficulty
     - Difficulty: `Math.sqrt(valleyX * valleyX + valleyZ * valleyZ) * 0.5 + rng[7] * 2` (harder further from home)
   - `generateValleyEnemySpawns(valley: ValleyDef): Vec3[]` -- positions within valley bounds
   - `generateValleyLoot(valley: ValleyDef): LootSpawn[]` -- loot items from seed
   - `generateValleyNPCs(valley: ValleyDef): NPCSpawn[]` -- NPC placements for settlement/merchant valleys

3. Create `src/world/LootSystem.ts`:
   - Item generation from `deriveValues` with salt `200000`
   - Rarity tiers with weighted selection (matches PuppetJSX rarity system)
   - `generateLootItem(x, z, worldSeed): ItemDef`
   - `applyItem(item: ItemDef, player: TankEntity): void` -- applies buff/heal

4. Export all functions. No React components in this step -- pure logic only.

**Verification**: Write a simple test in the browser console that calls `generateValley(0, 0, 54321)` and `generateValley(1, 0, 54321)` -- verify they produce different but deterministic results. Call each twice and confirm identical output.

---

### Step 2: Procedural Terrain (3D-Procedural-Terrain-V2-JSX)

**Reference**: Read `3D-Procedural-Terrain-V2-JSX/3D-Procedural-Terrain-V2-JSX-integration.md`

**Goal**: Create `src/world/TerrainSystem.ts` and `src/components/3d/Terrain.tsx` for infinite terrain.

**Actions**:

1. Create `src/world/TerrainSystem.ts`:
   - Port the `SeededRNG` and `SeededNoise` classes from `ProceduralTerrainV2.jsx` to TypeScript
   - Port the fractal noise function (multi-octave, persistence, lacunarity)
   - Port the biome color system (6 biomes with height-based vertex coloring)
   - Implement `getTerrainHeight(worldX: number, worldZ: number, worldSeed: number): number`:
     - Determine which valley the point is in: `valleyX = Math.floor(worldX / VALLEY_SIZE)`, same for Z
     - Get valley biome from `generateValley()`
     - If valley is home base `(0,0)`: return 0 (flat, structures from editor placed separately)
     - Compute base height from fractal noise using biome's `heightScale` and `noiseScale`
     - Add mountain ridges at valley borders:
       - `distToEdge = min distance to any valley edge`
       - If `distToEdge < VALLEY_PASS_WIDTH` and the relevant connection is true: no ridge (it's a pass)
       - Otherwise: `ridgeHeight = VALLEY_MOUNTAIN_HEIGHT * smoothstep(1 - distToEdge / BORDER_BLEND_WIDTH)`
     - Flatten under structures: terrain masking system (from ProceduralTerrainV2 grid mask)
   - Implement chunk manager:
     - `TerrainChunkManager` class
     - `updateChunks(playerX: number, playerZ: number)` -- activate/deactivate chunks based on distance
     - Each chunk: `{ worldX, worldZ, resolution, vertices, colors, needsUpdate }`
     - Resolution based on LOD distance constants
     - Max active chunks: `Math.ceil(RENDER_DISTANCE / CHUNK_SIZE)^2` (about 64)

2. Create `src/components/3d/Terrain.tsx`:
   - React Three Fiber component using `useFrame` for chunk streaming
   - Each chunk is a `<mesh>` with `BufferGeometry` (position + color attributes)
   - Wireframe overlay matching valley biome color (TRON aesthetic)
   - `<group>` contains all active chunk meshes
   - Frustum culling enabled per-chunk
   - Use `useMemo` to avoid regenerating static chunks

3. Update `src/game/Physics.ts`:
   - In movement update, call `getTerrainHeight(playerX, playerZ, worldSeed)` to get ground level
   - Tank Y position lerps toward terrain height (gravity when above, snap when below)
   - Slope detection from height samples at tank front/back for pitch, left/right for roll
   - Mountain collisions: if terrain height at next position exceeds `VALLEY_MOUNTAIN_HEIGHT * 0.7`, treat as wall -- apply wall-slide like existing arena walls
   - Tank stays grounded: `player.position.y = max(terrainHeight, player.position.y - gravity * dt)`

**Verification**: Load the game, drive the tank. Terrain should generate around you as you move. Mountains should block movement at valley edges except at passes. The home base area should be flat.

---

### Step 3: Home Base & Level Editor (Existing LevelEditorScreen)

**Goal**: Repurpose the Level Editor to build the player's home base at `(0, 0)`.

**Actions**:

1. Create `src/world/HomeBase.ts`:
   - `loadHomeBase(): ArenaDefinition` -- loads from `localStorage` key `rt-home-base`, or returns default
   - `saveHomeBase(arena: ArenaDefinition): void` -- saves to localStorage
   - Default home base: flat 500x500 area with 4 starting structures (walls, a garage, a comm tower, a supply depot) and 4 enemy spawns at the edges

2. Update `src/screens/LevelEditorScreen.tsx`:
   - Remove arena dropdown selector (only one arena: home base)
   - Change title to "Home Base Editor"
   - Save/load uses `rt-home-base` instead of `rt-custom-arenas`
   - Arena dimensions fixed to `VALLEY_SIZE x VALLEY_SIZE`
   - Player spawn fixed at `(0, 0, 0)`
   - Add "Return to Base" button that goes to `home_base` screen

3. Update `HomeBaseScreen.tsx` (rename from `TheBunkerScreen.tsx`):
   - Remove arena selection grid (no arenas to select)
   - Keep: tank color picker, armor slider, shield recharge slider
   - Add: wingman selector (4 wingman portraits with descriptions)
   - Add: "Edit Home Base" button -> level editor
   - Add: "Deploy" button -> open world screen
   - loadout stored in `rt-loadout` (existing key)
   - Selected wingman stored in loadout: `loadout.selectedWingman: number`

4. Update `GameState.ts`:
   - Replace `selectedArenaId` with `currentValley: ValleyDef | null`
   - Add `playerProgress: PlayerProgress` (persisted to `rt-progress`)
   - Add `worldSeed: number` (from constants or player-chosen)
   - Add `homeBase: ArenaDefinition` (loaded from HomeBase.ts)
   - Add `selectedWingman: number` (0-3)
   - Add `wingmanCooldown: number` (ms remaining)
   - Add `inventory: ItemDef[]`
   - Add `activeDialog: DialogTree | null`
   - Add `activeDialogNode: string | null`
   - Add actions: `enterValley`, `collectLoot`, `advanceDialog`, `useItem`, `callWingman`

---

### Step 4: SNES Audio SFX System (SNESaudio-JSX)

**Reference**: Read `SNESaudio-JSX/SNESaudio-integration.md`

**Goal**: Create `src/audio/SNESAudio.ts` -- synthesised 16-bit SFX for all game events.

**Actions**:

1. Port `SNESAudioEngine` class from `SNESSoundGenerator.jsx` to TypeScript in `src/audio/SNESAudio.ts`:
   - Constructor: no AudioContext yet (lazy init on first user interaction)
   - `init()`: create AudioContext, master gain, analyser, noise buffer, pulse wave
   - `resume()`: handle suspended context
   - `setVolume(v: number)`: master volume 0-1
   - `playPreset(preset: SNESPreset)`: full synthesis chain (oscillator/noise -> filter -> bitcrush -> envelope -> gain)
   - `playArpeggio(preset: SNESPreset)`: arpeggio sequence playback
   - TypeScript interface `SNESPreset` with all fields (waveform, baseFreq, freqSweep, duration, ADSR, filter, echo, bitCrush, etc.)

2. Port all 32 presets as `SNES_PRESETS` constant object, organised by category:
   - `movement`: jump, doubleJump, dash, land
   - `pickup`: coin, gem, heart, key
   - `combat`: laser, blaster, sword, punch, magic
   - `explosion`: explosion, pop, boom
   - `ui`: select, confirm, cancel, pause, text
   - `power`: powerUp, levelUp, oneUp, heal
   - `damage`: hurt, death, enemyHit, warning
   - `environment`: door, chest, splash, teleport, step

3. Create singleton accessor:
   ```typescript
   let _instance: SNESAudioEngine | null = null;
   export function getSNESAudio(): SNESAudioEngine {
     if (!_instance) _instance = new SNESAudioEngine();
     return _instance;
   }
   export function playSound(presetName: string): void {
     const engine = getSNESAudio();
     engine.init();
     engine.resume();
     const preset = SNES_PRESETS[presetName];
     if (preset) engine.playPreset(preset);
   }
   ```

4. Wire `playSound()` calls into existing systems:
   - `WeaponSystem.ts`: on fire -> `playSound('laser')` / `'explosion'` / `'blaster'` / `'punch'` / `'magic'` based on weapon type
   - `GameState.ts`: on enemy destroyed -> `playSound('boom')`
   - `GameState.ts`: on player damaged -> `playSound('hurt')`
   - `SecretSystem.ts`: on secret found -> `playSound('chest')`
   - Screen buttons: on click -> `playSound('select')`
   - Menu confirm: `playSound('confirm')`

5. Connect to existing `masterVolume` / `sfxVolume` settings in `GameState.ts`.

---

### Step 5: Engine Sounds (EngineSounds-JSX)

**Reference**: Read `EngineSounds-JSX/soundengine-integration.md`

**Goal**: Create `src/audio/EngineAudio.ts` -- continuous synthesised tank engine sound.

**Actions**:

1. Port `OutrunAudioEngine` class from `audio-engine.jsx` to TypeScript:
   - Multiple oscillators for engine harmonics
   - Exhaust noise (filtered noise buffer)
   - Tire/tread sound (speed-dependent noise)
   - Wind noise (high-pass filtered noise, volume from speed)
   - Master gain connected to AudioContext destination

2. Create adapter function `mapTankStateToAudio(player: TankEntity)`:
   ```typescript
   return {
     throttle: inputManager.isForwardHeld() ? 1.0 : 0.0,
     brake: inputManager.isBackwardHeld(),
     turning: (inputManager.isLeftHeld() ? -1 : 0) + (inputManager.isRightHeld() ? 1 : 0),
     speed: Math.min(1, magnitude(player.velocity) / FORWARD_SPEED),
     gear: Math.max(1, Math.min(5, Math.ceil(normalizedSpeed * ENGINE_GEARS))),
   };
   ```

3. Call `engineAudio.update(mappedState)` each frame in the game loop (after physics, before render).

4. Handle boost: when `player.isBoosting`, set throttle to 1.0 and trigger `triggerGearShift()`.

5. Handle pause: `engineAudio.stop()` on pause, `engineAudio.start()` on resume.

6. Handle game over: fade out and stop.

7. Connect to `masterVolume` / `sfxVolume` from settings.

---

### Step 6: Explosion System (ExplosionEngine-JSX)

**Reference**: Read `ExplosionEngine-JSX/Explosion-integration.md`

**Goal**: Create `src/combat/ExplosionManager.ts` and `src/components/3d/Explosions.tsx`.

**Actions**:

1. Port from `ExplosionEngine.jsx` to TypeScript:
   - `Explosion` component (particles, shockwave ring, flash, falling wreckage)
   - `useExplosionManager` hook (explosion lifecycle: trigger, update, remove)
   - `ExplosionRenderer` component (renders all active explosions)
   - `EXPLOSION_CONFIG` with classes: SMALL (fighters), MEDIUM (tanks), LARGE (elite), BOSS

2. Create `src/combat/ExplosionManager.ts`:
   - Object pool for explosions (max 20 concurrent)
   - `triggerExplosion(position: Vec3, class: 'SMALL'|'MEDIUM'|'LARGE'|'BOSS')`
   - Integrate with existing `effects` array in GameState (replace simple hit_flash/explosion with full explosion system)

3. Create `src/components/3d/Explosions.tsx`:
   - Uses `useExplosionManager` hook
   - Renders inside the Canvas alongside other 3D components
   - Explosion particles use `InstancedMesh` for performance (from zerobytes approach)
   - Custom TRON-palette particle colors: core=white/cyan, fire=magenta/orange, smoke=dark purple, sparks=yellow

4. Wire triggers:
   - Enemy tank destroyed -> MEDIUM explosion at enemy position
   - Player destroyed -> LARGE explosion
   - Plasma Mortar impact -> SMALL explosion at hit point
   - Wingman attacks -> MEDIUM explosions at targets
   - Boss enemy destroyed -> BOSS explosion with chain sequence

5. Camera shake: on MEDIUM+ explosions, apply shake to camera (existing `cameraShake` setting).

---

### Step 7: Laser Frame UI (LaserFrame-JSX)

**Reference**: Read `LaserFrame-JSX/README.md`

**Goal**: Create `src/components/ui/LaserFrameUI.tsx` and apply to all UI panels.

**Actions**:

1. Port from `laserprint-demo.jsx` to TypeScript:
   - `LaserBeam` component: SVG animated line from origin to target
   - `HologramPanel` component: orchestrates 4 beams to corners, then fill + content reveal
   - CRT scanline overlay

2. Create `src/components/ui/LaserFrameUI.tsx`:
   - `<LaserPanel>` wrapper component:
     ```tsx
     interface LaserPanelProps {
       x: number; y: number; width: number; height: number;
       color?: string; delay?: number; speed?: number;
       title?: string; children: React.ReactNode;
     }
     ```
   - Renders SVG container with `HologramPanel` inside
   - Accepts children as panel content (rendered after construction animation completes)
   - Color defaults to `COLORS.cyan`

3. Apply `<LaserPanel>` to:
   - `MainMenuScreen.tsx`: title card and menu buttons each in a laser panel, staggered delays
   - `HomeBaseScreen.tsx`: loadout panel, wingman selector panel, deploy button panel
   - `SettingsScreen.tsx`: each settings section in a panel
   - `ResultsScreen.tsx`: stats panels
   - Dialog boxes (DialogOverlay wraps in LaserPanel)
   - Inventory popup (when picking up loot)

4. Animation trigger: panels re-animate when screen mounts (key prop changes force remount).

5. TRON palette integration: panels use `COLORS.cyan` for primary, `COLORS.magenta` for alerts, `COLORS.orange` for combat info.

---

### Step 8: Attitude Indicator HUD (3D-attitude-indicator-JSX)

**Reference**: Read `3D-attitude-indicator-JSX/attitude-integration-guide.md`

**Goal**: Create `src/components/ui/AttitudeHUD.tsx` -- orientation display in the game HUD.

**Actions**:

1. Port `AttitudeIndicatorCanvas.jsx` to TypeScript as `src/components/ui/AttitudeHUD.tsx`:
   - Canvas 2D renderer (not Three.js -- performance-friendly HTML overlay)
   - Modify color palette to match TRON theme:
     ```typescript
     const COLORS_TRON = {
       sky: '#0a2e4a',
       skyDark: '#051a2e',
       ground: '#1a0a2e',
       groundDark: '#0a051a',
       horizon: '#00FFFF',
       bezel: '#0a0a0a',
       textGreen: '#00FF00',
       textCyan: '#00FFFF',
       textMagenta: '#FF00FF',
       textWhite: '#FFFFFF',
       textYellow: '#FFD700',
       aircraft: '#FF8C00',
       slipBall: '#FFFFFF',
     };
     ```

2. Map tank state to indicator props:
   ```typescript
   const indicatorProps = {
     pitch: player.driftAngle * 10,   // visual pitch from terrain slope
     roll: calculateTerrainRoll(player.position),
     heading: (player.rotation * 180 / Math.PI + 360) % 360,
     airspeed: Math.round(magnitude(player.velocity)),
     altitude: Math.round(getTerrainHeight(player.position.x, player.position.z, worldSeed)),
     verticalSpeed: Math.round(player.velocity.y * 60),
     slip: player.isDrifting ? (player.driftAngle / Math.PI) : 0,
     waypoint: currentValley?.id === 'home' ? 'HOME' : `V${currentValley?.gridX},${currentValley?.gridZ}`,
     distance: Math.round(magnitude(player.position)),  // distance from (0,0)
     size: 150,
   };
   ```

3. Position: fixed bottom-right of screen, `pointer-events: none`.

4. Performance: throttle updates to 30 Hz (every other frame) using `useRef` counter.

5. Respect `showFps` setting -- if HUD is hidden, hide indicator too.

---

### Step 9: RPG Dialog System (RPGdialog-JSX)

**Reference**: Read `RPGdialog-JSX/RPG-chat-integration.md`

**Goal**: Create `src/components/ui/DialogOverlay.tsx` and `src/data/npcDialogs.ts`.

**Actions**:

1. Port `RPGDialogue.jsx` to TypeScript as `src/components/ui/DialogOverlay.tsx`:
   - Support `mode: 'toast' | 'fullscreen'`
   - Typewriter text effect
   - Animated portrait (3 frames for mouth animation)
   - `onComplete` callback
   - `useDialogueSequence` hook for multi-step conversations

2. Create `src/data/npcDialogs.ts` -- dialog trees for each NPC type:
   - **Merchant**: "Welcome, commander. Need supplies?" -> Buy items / Sell items / Leave
   - **Mechanic**: "Your tank looks battle-worn. Repairs?" -> Repair (costs loot) / Upgrade armor / Leave
   - **Scout**: "I've surveyed the nearby valleys." -> Reveal map (marks valleys on minimap) / Ask about enemies / Leave
   - **Quest Giver**: "Commander, we need your help." -> Accept quest / Decline / Ask for details
   - Each tree is a `DialogTree` with `nodes: Record<string, DialogNode>` and `startNodeId`

3. Create `src/world/NPCSystem.ts`:
   - NPCs generated per-valley using `deriveValues` with salt `150000`
   - Settlement valleys: 2-4 NPCs (merchant guaranteed)
   - Other valleys: 0-1 NPC (scout or quest giver, rare)
   - NPC names generated from seed (array of name parts combined)
   - Interaction: when player within `NPC_INTERACT_RADIUS` and presses `E`, dialog opens

4. Wire into `GameState.ts`:
   - `activeDialog` / `activeDialogNode` state
   - `openDialog(tree: DialogTree)`: sets active dialog, pauses game
   - `advanceDialog(choiceIndex?: number)`: progresses to next node or closes
   - `closeDialog()`: clears active dialog, resumes game

5. Portrait generation: since we have no image assets, NPCs use colored placeholder rectangles with procedurally-generated "face" patterns (simple Canvas 2D drawing based on NPC seed). Three frames with mouth position variants.

6. Add `E` key binding in `Input.ts` for NPC interaction.

---

### Step 10: Wingman Air Support (WingmanSupport-JSX)

**Reference**: Read `WingmanSupport-JSX/TeamSpecial-integration.md`

**Goal**: Create `src/combat/WingmanSystem.ts` and `src/components/3d/WingmanShip.tsx`.

**Actions**:

1. Port core logic from `WingmanSupport.jsx` to TypeScript:
   - `WingmanSystem` class in `src/combat/WingmanSystem.ts`:
     - State: `selectedWingman`, `cooldownRemaining`, `isActive`, `phase`
     - `canActivate()`: returns true if cooldown is 0 and not active
     - `activate(playerPosition: Vec3, playerForward: Vec3, enemies: TankEntity[])`: begins wingman sequence
     - `update(dt: number)`: advances phase (approach -> attack -> depart), updates cooldown
     - Phases:
       1. Approach (1s): ship flies in from behind player
       2. Attack (3s): attack run based on wingman type
       3. Depart (1s): ship flies away
     - On attack: damage enemies in target area based on type
   - Attack implementations:
     - **Gatling**: 8 rapid hits on enemies in a 30-degree cone ahead, 15 damage each
     - **Beam**: continuous sweep across 60 degrees, 50 damage to all hit
     - **Lightning**: chains from nearest enemy to up to 5 others within 20 units, 30 damage each
     - **Missile**: fires 4 homing projectiles at 4 nearest enemies, 90 damage each

2. Create `src/components/3d/WingmanShip.tsx`:
   - Simple wireframe ship mesh (box + triangles for wings, TRON style)
   - Animated along flight path (approach from behind + above -> dive toward enemies -> pull up and away)
   - Point light matching wingman color
   - Trail effect (line from previous positions)
   - Attack effects per type:
     - Gatling: small orange tracers
     - Beam: cyan line from ship to target
     - Lightning: branching magenta lines
     - Missile: green projectile meshes with smoke trails

3. Create `src/components/ui/WingmanHUD.tsx`:
   - Shows 4 wingman icons (colored squares with name labels)
   - Selected wingman highlighted with glow
   - Cooldown bar below (fills from left to right)
   - "READY" text when available, seconds remaining when on cooldown
   - Keys `6-9` to select, `Q` to activate
   - Positioned top-right of screen

4. Wire into game loop:
   - `Input.ts`: add `Q` key binding for wingman activation, `6-9` for selection
   - `OpenWorldScreen.tsx` game loop: call `wingmanSystem.update(dt)` each frame
   - On activation: trigger explosion effects at each enemy hit position
   - Play `playSound('powerUp')` on activation

---

### Step 11: Large-Scale Combat Layer (zerobytes-3dcombatlayer)

**Reference**: Read `zerobytes-3dcombatlayer/ZB-3DCombatLayer-integration.md`

**Goal**: Create `src/combat/CombatLayer.ts` -- efficient combat resolution for world-scale battles.

**Actions**:

1. Port core math from `ZB-3DCombatLayerV3.jsx` to TypeScript in `src/combat/CombatLayer.ts`:
   - `positionHash(x, y, z, salt)`: deterministic 3D hash
   - `coherentValue3D(x, y, z, seed, octaves, frequency)`: smooth 3D noise
   - `generateUnit(x, y, z, worldSeed, faction)`: create unit from position
   - `resolveAttack3D(attacker, target, attackId, worldSeed)`: single attack resolution
   - `resolveBattleRound(attackers, defenders, worldSeed, attackId)`: full round

2. Adapt for R-TANKS 5 context:
   - "Units" are enemy tanks with stats derived from valley difficulty
   - Faction 0 = player allies (home base defenders), Faction 1 = enemy forces
   - Unit types map to tank variants: Scout (fast, low armor), Heavy (slow, high armor), Standard, Artillery (long range), Elite
   - Attack resolution uses tank weapon stats (not generic fantasy weapons)

3. Create `src/combat/CombatLayer.ts` integration:
   - `resolveValleyCombatState(valley: ValleyDef, worldSeed: number): CombatResult`
     - Computes how many enemies in a valley are still alive based on seed
     - As time progresses (game time), additional "rounds" resolve
     - Returns: alive enemy positions, destroyed enemy positions (for wreckage), battle intensity (for distant VFX)
   - `getDistantBattleEffects(playerPos: Vec3, valleys: ValleyDef[]): BattleEffect[]`
     - For valleys beyond direct simulation range, returns instanced particle data
     - Each effect: position, intensity, color -- rendered as instanced flash meshes

4. Create `src/components/3d/DistantBattles.tsx`:
   - Uses `InstancedMesh` with ~100 instances
   - Each instance: small sphere with emissive material, pulsing opacity
   - Position from `getDistantBattleEffects()`
   - Updates at 2 Hz (low frequency, just ambient visual)
   - Colors: orange for active combat, red for heavy combat, dim yellow for skirmishes

5. Integration with direct combat:
   - When player enters a valley, `resolveValleyCombatState()` determines which seed-enemies are already dead
   - Surviving enemies transition to full AI simulation (Tier 1 combat)
   - When player leaves, surviving AI enemies are "frozen" (their count is recorded)

---

### Step 12: Open World Game Loop (OpenWorldScreen)

**Goal**: Create `src/screens/OpenWorldScreen.tsx` -- the main game screen replacing `BattleTanksScreen.tsx`.

**Actions**:

1. Create `src/screens/OpenWorldScreen.tsx`:
   - `useFrame` game loop implementing the 11-step pipeline described in Functionality
   - Terrain chunk streaming managed by `TerrainChunkManager`
   - Valley transition detection: when player crosses valley boundary, call `enterValley(newValley)`
   - Enemy management:
     - Current valley + adjacent valleys: full AI entities (max 20 total from `MAX_ENEMIES`)
     - Other valleys: seed-resolved combat state
   - Pause menu with "Return to Home Base" option (teleport player to `(0,0,0)`)

2. Camera system:
   - Retain existing third-person camera from R-TANKS 4
   - Camera height adapts to terrain: `CAMERA_HEIGHT` above terrain at player position
   - Camera lerp (existing `CAMERA_LERP`) follows player smoothly over hills

3. HUD overlay (HTML layer over Canvas):
   - Health bar, shield bar (existing)
   - Weapon selector (existing, keys 1-5)
   - Wingman HUD (new, top-right)
   - Attitude indicator (new, bottom-right)
   - Minimap (new, top-left) -- shows valley grid, player dot, enemy dots in current valley
   - Valley name banner (appears for 3 seconds on valley entry)
   - FPS display (existing)
   - Loot pickup prompt ("Press F to pick up [Item Name]")
   - NPC interact prompt ("Press E to talk to [NPC Name]")

4. Minimap implementation (`src/components/ui/WorldMapHUD.tsx`):
   - 150x150px canvas in top-left
   - Shows 5x5 grid of valleys centered on player
   - Each valley cell colored by type: green=settlement, red=combat, yellow=treasure, purple=boss, gray=empty, orange=hazard
   - Player shown as white dot
   - Visited valleys have full color, unvisited are dim
   - Mountain passes shown as gaps in cell borders

5. Valley enter effects:
   - Display valley name and biome in LaserFrame banner for 3 seconds
   - Play `playSound('teleport')` on valley transition
   - Trigger terrain chunk regeneration for new biome

---

### Step 13: Main Menu & Screen Flow Updates

**Goal**: Update all screen transitions for the open world flow.

**Actions**:

1. **MainMenuScreen.tsx**:
   - Title: "R-TANKS 5" (cyan), subtitle: "Open World" (magenta)
   - Buttons: "New Game", "Continue" (if save exists), "Settings"
   - All wrapped in LaserFrame panels
   - "New Game": prompt for world seed (or random), then go to `home_base`
   - "Continue": load `rt-progress` from localStorage, go to `open_world` at saved position
   - Background: slowly rotating tank model (retained from R-TANKS 4)

2. **HomeBaseScreen.tsx** (formerly TheBunkerScreen):
   - Loadout customisation (retained)
   - Wingman selection (new)
   - "Edit Home Base" -> level editor
   - "Deploy" -> open world at `(0, 0, 0)`
   - "Continue from Last Position" -> open world at saved position (if different from home)

3. **ResultsScreen.tsx**:
   - Updated stats: valleys explored, enemies destroyed, loot collected, distance traveled, play time
   - "Return to Home Base" button
   - Stats saved to `rt-progress`

4. **SettingsScreen.tsx**:
   - New sliders: Engine Sound Volume, SFX Volume (separate from existing master/music/sfx)
   - New toggle: Show Minimap
   - New toggle: Show Attitude Indicator
   - Retained: camera mode, camera shake, color blind mode, high contrast, FPS display, secret hints

5. **App.tsx**:
   - Update screen router for new screen names
   - Handle `home_base`, `open_world`, `level_editor`, `settings`, `results`, `main_menu`

---

### Step 14: Save System & Persistence

**Goal**: Implement save/load for open world progress.

**Actions**:

1. **localStorage keys**:
   - `rt-settings` -- retained (game settings)
   - `rt-loadout` -- retained + `selectedWingman` field
   - `rt-home-base` -- new (home base ArenaDefinition)
   - `rt-progress` -- new (PlayerProgress: position, visited valleys, collected loot, inventory, kills, playtime)
   - `rt-secret-progress` -- updated (secrets now keyed by valley coords, not arena id)

2. **Auto-save**: every 30 seconds while in open world, save current `PlayerProgress` to `rt-progress`.

3. **Manual save**: pause menu "Save Game" button.

4. **Load**: on "Continue" from main menu, restore full state.

5. **Reset**: "New Game" clears `rt-progress` and `rt-home-base` (with confirmation dialog).

---

### Step 15: Integration Testing & Polish

**Goal**: End-to-end testing and visual polish pass.

**Actions**:

1. **Type checking**: `npx tsc --noEmit` -- fix all TypeScript errors.

2. **Build test**: `npm run build` -- verify production build succeeds and is under 2MB.

3. **Gameplay test checklist**:
   - [ ] Player spawns at home base `(0, 0)`
   - [ ] Tank drives over terrain with proper height following
   - [ ] Mountains block movement at valley edges
   - [ ] Mountain passes allow crossing to adjacent valleys
   - [ ] Valley content generates correctly from seed (different biomes, enemy counts)
   - [ ] All 5 weapons fire and deal damage
   - [ ] SNES SFX play for all mapped events
   - [ ] Engine sound responds to throttle/speed/boost
   - [ ] Enemies spawn in combat valleys and use AI states
   - [ ] Loot items appear and can be picked up
   - [ ] NPCs appear in settlements and dialog system works
   - [ ] Wingman can be called and attacks enemies
   - [ ] Explosions play on enemy death
   - [ ] Attitude indicator shows correct heading/speed
   - [ ] Minimap shows valley grid and player position
   - [ ] LaserFrame effect animates on screen transitions
   - [ ] Save/load preserves progress
   - [ ] Home base editor works and structures persist
   - [ ] Level editor structures appear at home base in open world
   - [ ] Distant battles show instanced particle effects
   - [ ] Performance: 60 FPS on mid-range hardware

4. **Polish**:
   - Smooth camera transitions on valley entry
   - Fade-in terrain chunks to avoid pop-in
   - Ensure all TRON neon colors are consistent across systems
   - Add brief screen flash on player damage
   - Valley entry chime (SNES `teleport` preset)
   - Boss valley warning banner with red LaserFrame

---

## Data Models

### Valley Grid Coordinate System

```
         N (valleyZ - 1)
         |
W ---- (vX, vZ) ---- E
         |
         S (valleyZ + 1)

World position to valley: valleyX = Math.floor(worldX / 500)
Valley to world origin: worldX = valleyX * 500

Home base: valleyX = 0, valleyZ = 0
Difficulty increases with distance from (0, 0)
```

### Terrain Height Pipeline

```
worldX, worldZ
     |
     v
valleyX, valleyZ = floor(world / VALLEY_SIZE)
     |
     v
valley = generateValley(valleyX, valleyZ, worldSeed)
     |
     v
biome = valley.biome -> BIOMES[biome] config
     |
     v
baseHeight = fractalNoise(worldX * noiseScale, worldZ * noiseScale, octaves) * heightScale
     |
     v
borderDist = minDistToValleyEdge(worldX, worldZ)
     |
     v
if borderDist < blendWidth AND no connection on that side:
  ridgeHeight = MOUNTAIN_HEIGHT * smoothstep(...)
     |
     v
if at home base (0,0): return 0 (flat) + editor structure heights
     |
     v
finalHeight = baseHeight + ridgeHeight
```

### Combat Resolution Pipeline (Seeded)

```
valley position (vX, vZ) + worldSeed
     |
     v
valleySeed = coordsToSeed(vX, vZ, 0, worldSeed)
     |
     v
enemies = generateValleyEnemies(valleySeed, difficulty, count)
     |
     v
if player in valley -> Tier 1: full AI simulation
if player nearby   -> Tier 2: seeded resolution per round
if player far      -> frozen state, instanced VFX only
```

---

## Performance Budget

| System | Target | Approach |
|--------|--------|----------|
| Terrain chunks | <16ms generation | Web Worker for noise, typed arrays |
| Active enemies | Max 20 | Object pool, 10 Hz AI updates |
| Explosions | Max 20 concurrent | Object pool, instanced particles |
| Distant battles | Max 100 instances | InstancedMesh, 2 Hz updates |
| Audio | <2ms per frame | Pre-computed noise buffer, minimal oscillators |
| HUD | <4ms per frame | Canvas 2D (attitude), SVG (laser), HTML (text) |
| Total frame | <16.67ms (60 FPS) | Budget: 8ms game logic, 8ms render |

---

## Migration Checklist (R-TANKS 4 -> R-TANKS 5)

- [x] Retain: Tank entity model, 5 weapons, physics, boost/drift
- [x] Retain: Zustand state management pattern
- [x] Retain: @react-three/fiber rendering pipeline
- [x] Retain: Input manager (extended with new keys)
- [x] Retain: Camera system (updated for terrain following)
- [x] Retain: Settings screen and localStorage persistence
- [ ] Remove: 10 hardcoded arenas (`src/data/arenas.ts`)
- [ ] Remove: `TheBunkerScreen.tsx` (replaced by `HomeBaseScreen.tsx`)
- [ ] Remove: `BattleTanksScreen.tsx` (replaced by `OpenWorldScreen.tsx`)
- [ ] Replace: Arena collision walls -> terrain mountain collision
- [ ] Replace: Fixed enemy spawns -> seeded world spawns
- [ ] Replace: Arena-bound secrets -> valley-seeded secrets
- [ ] Add: Infinite procedural terrain
- [ ] Add: Valley generation system
- [ ] Add: SNES audio engine
- [ ] Add: Engine sound synthesis
- [ ] Add: Explosion engine
- [ ] Add: LaserFrame UI
- [ ] Add: Attitude indicator HUD
- [ ] Add: RPG dialog system
- [ ] Add: Wingman air support
- [ ] Add: Large-scale combat layer
- [ ] Add: Loot/item system
- [ ] Add: NPC system
- [ ] Add: Save/load system
- [ ] Add: Minimap
