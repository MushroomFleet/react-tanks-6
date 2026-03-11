# SpacetimeDB Multiplayer Integration Plan
## Project: REACT-TANKS (Car-Mero Enhanced Edition)
> Stack: React 18 + TypeScript + Vite + @react-three/fiber + drei + three.js + zustand
> Client Mode: React (R3F + Zustand hybrid)
> Generated: 2026-03-05

---

## Executive Summary

REACT-TANKS is a TRON-inspired wireframe tank combat game built on a **ZeroBytes position-as-seed architecture** — the entire world (terrain, valleys, biomes, loot, NPCs, enemy spawns, combat resolution) is deterministically derived from coordinates + a single `worldSeed` integer. This dramatically simplifies multiplayer: the server never stores world state, only **delta events** (what changed — kills, loot collected, secrets found). Multiplayer integration enables real-time PvP tank combat, cooperative open-world exploration, and persistent identity. The plan stages from foundation (Phase 1) through presence (Phase 2), ZeroBytes-aware game system sync (Phase 3), server-authoritative validation (Phase 4), and persistent identity (Phase 5).

---

## Codebase Assessment

### Detected Stack

| Dependency | Version |
|---|---|
| react | ^18.2.0 |
| react-dom | ^18.2.0 |
| @react-three/fiber | ^8.15.0 |
| @react-three/drei | ^9.88.0 |
| three | ^0.158.0 |
| zustand | ^4.4.0 |
| uuid | ^9.0.0 |
| vite | ^5.0.0 |
| typescript | ^5.2.0 |

### State Inventory

| State | Current Location | Multiplayer Scope | Priority |
|---|---|---|---|
| player.position / rotation / velocity | `GameState.ts` zustand `player: TankEntity` | **Shared** — all players see each other | Phase 1 |
| player.health / shield | `GameState.ts` zustand `player` | **Server-authoritative** — prevents health hacking | Phase 4 |
| player.currentWeapon / weaponCooldowns | `GameState.ts` zustand `player` | **Shared** — visible weapon indicator | Phase 2 |
| player.isBoosting / isDrifting | `GameState.ts` zustand `player` | **Shared** — visual trails for other players | Phase 2 |
| player.turretRotation | `GameState.ts` zustand `player` | **Shared** — turret aiming visible to others | Phase 2 |
| enemies (AI tanks) | `GameState.ts` zustand `enemies: TankEntity[]` | **ZeroBytes-derived spawns** — spawns deterministic from valley coords; only kill-state synced. Arena mode: server runs AI | Phase 3 |
| lingerZones | `GameState.ts` zustand `lingerZones: LingerZone[]` | **Server-authoritative** — devastator DOT zones (PvP only) | Phase 3 |
| secrets (discovered state) | `GameState.ts` + localStorage `rt-secret-progress` | **ZeroBytes-derived positions** — only `discoveredSecrets: Set<string>` synced | Phase 3 |
| effects (muzzle_flash, tracer, etc.) | `GameState.ts` zustand `effects[]` | **Local-only** — derived from events, rendered client-side | N/A |
| gameTime / isPaused / isGameOver | `GameState.ts` zustand | **Server-authoritative** — server controls match state | Phase 3 |
| shotsFired / shotsHit / damageTaken | `GameState.ts` zustand | **Server-authoritative** — stats tracked server-side | Phase 4 |
| loadout (armor, shieldRecharge, color) | `GameState.ts` zustand + localStorage `rt-loadout` | **Per-player** — sent to server on connect | Phase 1 |
| settings (camera, volume, etc.) | `GameState.ts` zustand + localStorage `rt-settings` | **Local-only** — never shared | N/A |
| customArenas | `GameState.ts` zustand + localStorage `rt-custom-arenas` | **Local-only** (future: shared arena hosting) | N/A |
| currentScreen | `GameState.ts` zustand | **Local-only** — UI routing | N/A |
| inventory / playerProgress | `GameState.ts` zustand + localStorage `rt-progress` | **Server-authoritative** — persistent progression. Loot items themselves are ZeroBytes-derived from position; only collected IDs + inventory list need sync | Phase 5 |
| nemesisState | `GameState.ts` zustand + localStorage | **Shared delta-only** — records + turn counter synced; roaming positions derived from `coordsToSeed(id, turn, worldSeed)` | Phase 3 |
| vehicleMode / parkedTank | `GameState.ts` zustand | **Shared** — others see on-foot vs in-tank | Phase 2 |
| activeDialog | `GameState.ts` zustand | **Local-only** — NPC dialogs are local UI | N/A |
| wingman selection/cooldown | `GameState.ts` zustand | **Shared** — wingman strikes visible to all | Phase 3 |
| camera shake / audio | Refs + singletons | **Local-only** — triggered by events | N/A |

### Game Loop Pattern

The game uses **React Three Fiber's `useFrame`** inside `<Canvas>` for the main loop:
- `GameLoop` component in `BattleTanksScreen.tsx:227` runs `useFrame` with capped delta (50ms max)
- `OpenWorldScreen.tsx` has a parallel `useFrame` loop with terrain, valley transitions, NPCs, loot
- AI updates are throttled to 10 Hz via `aiTimerRef`
- Input is sampled once per frame from the singleton `inputManager`
- Physics, weapon fire, damage, and effects are all processed in-frame
- State mutations go through `useGameStore.getState()` (direct Zustand access, not hooks — **good for multiplayer**, avoids re-render storms)

**Critical insight:** The game loop already uses `useGameStore.getState()` imperatively rather than hook-based subscriptions for high-frequency state. This means SpacetimeDB callbacks can write directly to Zustand without triggering re-renders on the game loop path.

### ZeroBytes: Position-as-Seed Deterministic Architecture

**This is the single most important architectural feature for multiplayer.** The codebase implements a "ZeroBytes" paradigm: *the coordinate IS the seed*. Every procedural element is computable in O(1) from coordinates + a shared `worldSeed` (constant `54321`). This means huge swathes of game state require **zero replication** — any client with the same `worldSeed` independently generates identical results.

#### How It Works

The foundation is `src/world/WorldSeed.ts`:

| Function | Purpose |
|---|---|
| `mulberry32(seed)` | Deterministic PRNG from integer seed |
| `coordsToSeed(x, y, z, salt)` | xxHash-inspired mixing of coordinates → 32-bit seed |
| `hashFloat(x, y, z, salt)` | Deterministic `[0,1)` float from any 3D position |
| `deriveValues(x, y, z, salt, count)` | N deterministic floats from position |
| `coherentValue(x, y, seed, octaves)` | Smoothly interpolated noise for regional properties |
| `weightedSelect(options, weights, randomValue)` | Deterministic weighted selection |

Every salt offset is unique per system (e.g., `worldSeed + 100000` for valleys, `+ 150000` for enemy spawns, `+ 200000` for loot, `+ 300000` for NPCs, `+ 400000` for landmarks, `+ 500000` for secrets, etc.), guaranteeing no cross-system collision.

#### Systems Built on Position-as-Seed

| System | File | What's Deterministic | Multiplayer Implication |
|---|---|---|---|
| **Valley generation** | `ValleyGenerator.ts` | Type, biome, difficulty, enemy count, NPC spawns, landmarks, secrets, mountain pass connections — all from `(valleyX, valleyZ, worldSeed)` | **Zero replication** — every client generates identical valleys |
| **Terrain height** | `TerrainSystem.ts` | `getTerrainHeight(worldX, worldZ, worldSeed)` — seeded simplex noise + fractal octaves + biome config + mountain ridges | **Zero replication** — terrain is identical on all clients; physics (ground snapping, mountain walls) need not be synced |
| **Loot items** | `LootSystem.ts` | `generateLootItem(x, z, worldSeed)` — rarity, type, stats, value all from `floor(x), floor(z)` | **Only sync collected state** — item definition is deterministic from position |
| **Valley loot spawns** | `LootSystem.ts` | `generateValleyLoot(valleyX, valleyZ, worldSeed, ...)` — spawn positions + items | **Only sync which IDs collected** |
| **Enemy drops** | `LootSystem.ts` | `generateEnemyDrop(enemyPos, worldSeed, killIndex)` — item from enemy death position + kill counter | **Only sync killIndex** — both clients derive identical drop |
| **Combat layer** | `CombatLayer.ts` | `resolveValleyCombatState(valley, worldSeed, gameTime, playerKills)` — full army generation, attack resolution, hit/crit rolls from `positionHash` | **Only sync gameTime + playerKills** — entire battle replays identically |
| **Distant battle VFX** | `CombatLayer.ts` | `getDistantBattleEffects(playerPos, valleys, worldSeed, gameTime)` — flash positions, colors, intensity | **Zero replication** — cosmetic, derived from shared gameTime |
| **Nemesis roaming** | `NemesisSystem.ts` | `advanceNemesisRoaming` — position from `coordsToSeed(hashStringToInt(r.id), newTurn, 0, worldSeed + 800000)` | **Only sync turn counter + nemesis records** |
| **Nemesis names** | `NemesisSystem.ts` | `generateNemesisName(seed)` — deterministic from integer seed | **Zero replication** |
| **NPC spawns** | `ValleyGenerator.ts` | Type, name, position from `deriveValues(valleyX, valleyZ, 0, worldSeed + 300000, 20)` | **Zero replication** |
| **Biome assignment** | `ValleyGenerator.ts` | `getValleyBiome(vx, vz, worldSeed)` — coherent noise → biome | **Zero replication** |
| **Terrain coloring** | `TerrainSystem.ts` | Vertex colors from biome config + noise value | **Zero replication** |

#### The Five ZeroBytes Laws (from `CombatLayer.ts`)

1. **O(1) Access** — `positionHash` gives any result instantly, no iteration
2. **Parallelism** — units/valleys/items generated independently, no ordering dependency
3. **Coherence** — noise functions for smooth regional properties (biomes, difficulty)
4. **Hierarchy** — `Battle > Valley > Unit > Attack` seeding chain prevents collisions
5. **Determinism** — same inputs = same outputs on any machine, any time

#### What This Means for the Multiplayer Schema

The SpacetimeDB schema becomes **dramatically thinner** than a naive approach:

| What the original plan stored server-side | What ZeroBytes actually needs |
|---|---|
| Full AI enemy table with positions, health, AI state | Just `worldSeed` + `enemyKills: Set<string>` per valley |
| Loot spawn definitions with item stats | Just `collectedLoot: Set<string>` |
| Valley terrain, biome, landmarks | Just `worldSeed` (single u32) |
| Secret definitions and positions | Just `discoveredSecrets: Set<string>` |
| Distant battle effect positions | Nothing — derived from `gameTime` |
| NPC positions, names, types | Nothing — derived from valley coordinates |
| Combat layer army composition | Just `gameTime` + `playerKills` per valley |

**The server needs to replicate only delta events (what changed), never world state (what exists).** Both the client and the server can independently reconstruct the full world from `worldSeed` at any time.

### Integration Complexity: **Medium** (revised from High)

Justification:
- ZeroBytes eliminates replication for world generation, terrain, loot definitions, NPC spawns, combat layer, and distant battle VFX
- Only 12 of 74 TankEntity fields need networking (position, rotation, turret, velocity, weapon, boost, drift, vehicle mode, color, health, shield, alive)
- AI enemies in open world need only kill-state sync (spawn positions are deterministic); server validates kills but doesn't need to run AI tick
- Arena mode AI is the main complexity: server must run AI state machine for shared PvE enemies (but only in arena mode, not open world)
- Hitscan combat + PvP damage still needs server validation (unchanged)
- The deterministic combat layer (`CombatLayer.ts`) means distant/unvisited valley combat resolves identically on all clients from shared `gameTime`

---

## Prerequisites

```bash
# Install SpacetimeDB CLI (Windows — PowerShell)
iwr https://install.spacetimedb.com/install.ps1 -useb | iex

# Or via npm
npm install -g @spacetimedb/cli

# Install client SDK
npm install spacetimedb

# Initialize dev environment
spacetime init --lang typescript spacetimedb
```

**Server module language recommendation:** TypeScript 2.0 — the team already works in TypeScript, the game logic (damage formulas, weapon stats, AI) can be shared between client and server modules with minimal translation. Only consider Rust if the server module exceeds Maincloud free-tier performance limits under load testing.

---

## Phase 1: Foundation — Single Player on SpacetimeDB

*Goal: Replace the local player's position/rotation/loadout with SpacetimeDB-backed state. Game still feels singleplayer. No gameplay changes.*

### 1.1 Define the Server Module

```typescript
// spacetimedb/src/index.ts
import { schema, t, table, SenderError } from 'spacetimedb/server';

// === TABLES ===

// Core player table — minimal high-frequency fields only
const player = table(
  { name: 'player', public: true },
  {
    identity: t.identity().primaryKey(),
    name: t.string(),
    x: t.f32(),
    y: t.f32(),
    z: t.f32(),
    rotation: t.f32(),
    turretRotation: t.f32(),
    velocityX: t.f32(),
    velocityZ: t.f32(),
    health: t.f32(),
    maxHealth: t.f32(),
    shield: t.f32(),
    maxShield: t.f32(),
    currentWeapon: t.string(),    // WeaponType
    isBoosting: t.bool(),
    isDrifting: t.bool(),
    vehicleMode: t.string(),      // 'tank' | 'on_foot'
    color: t.string(),
    armor: t.u8(),
    shieldRecharge: t.u8(),
    alive: t.bool(),
    arenaId: t.string(),          // which arena/valley the player is in
    score: t.u32(),
  }
);

// Match state — one row per active match
const matchState = table(
  { name: 'match_state', public: true },
  {
    id: t.u32().primaryKey().autoInc(),
    arenaId: t.string(),
    gameTime: t.f32(),
    isPaused: t.bool(),
    isGameOver: t.bool(),
    isVictory: t.bool(),
    maxPlayers: t.u8(),
    createdAt: t.u64(),
  }
);

// ZeroBytes world config — shared once, all clients derive everything from this
const worldConfig = table(
  { name: 'world_config', public: true },
  {
    id: t.u32().primaryKey(),       // always 1 (singleton)
    worldSeed: t.u32(),             // the single number that generates the entire world
    gameTime: t.f32(),              // server-authoritative elapsed time (drives CombatLayer)
  }
);

// Delta-only tables — ZeroBytes means we store WHAT CHANGED, not WHAT EXISTS
// Loot items, secrets, enemy stats are all derived from worldSeed + position.
// We only track which ones have been collected/discovered/killed.

const collectedLoot = table(
  { name: 'collected_loot', public: true },
  {
    id: t.string().primaryKey(),    // loot spawn ID (e.g. "loot_1_2_3")
    collectorIdentity: t.identity(),
    collectedAt: t.u64(),
  }
);

const discoveredSecret = table(
  { name: 'discovered_secret', public: true },
  {
    id: t.string().primaryKey(),    // secret ID (e.g. "secret_1_2_0")
    discovererIdentity: t.identity(),
    discoveredAt: t.u64(),
  }
);

const enemyKill = table(
  { name: 'enemy_kill', public: true },
  {
    id: t.string().primaryKey(),    // enemy ID (e.g. "enemy-valley_1_0-3")
    killerIdentity: t.identity(),
    valleyId: t.string(),           // which valley (for CombatLayer playerKills count)
    killedAt: t.u64(),
  }
);

const spacetimedb = schema({
  player, matchState, worldConfig,
  collectedLoot, discoveredSecret, enemyKill,
});
export default spacetimedb;

// === LIFECYCLE ===

export const onConnect = spacetimedb.clientConnected(ctx => {
  const existing = ctx.db.player.identity.find(ctx.sender);
  if (!existing) {
    ctx.db.player.insert({
      identity: ctx.sender,
      name: 'TANK-' + ctx.sender.toHexString().slice(0, 6).toUpperCase(),
      x: 0, y: 0, z: 0,
      rotation: 0, turretRotation: 0,
      velocityX: 0, velocityZ: 0,
      health: 100, maxHealth: 100,
      shield: 50, maxShield: 50,
      currentWeapon: 'precision',
      isBoosting: false, isDrifting: false,
      vehicleMode: 'tank',
      color: '#00FFFF',
      armor: 5, shieldRecharge: 5,
      alive: true,
      arenaId: '',
      score: 0,
    });
  }
});

export const onDisconnect = spacetimedb.clientDisconnected(ctx => {
  ctx.db.player.identity.delete(ctx.sender);
});

// === REDUCERS ===

// High-frequency: player position/state update (called ~20/sec)
export const updatePosition = spacetimedb.reducer(
  {
    x: t.f32(), y: t.f32(), z: t.f32(),
    rotation: t.f32(), turretRotation: t.f32(),
    velocityX: t.f32(), velocityZ: t.f32(),
    currentWeapon: t.string(),
    isBoosting: t.bool(), isDrifting: t.bool(),
    vehicleMode: t.string(),
  },
  (ctx, args) => {
    const p = ctx.db.player.identity.find(ctx.sender);
    if (!p) throw new SenderError('Player not found');
    ctx.db.player.identity.update({ ...p, ...args });
  }
);

// Loadout: sent once on match join
export const setLoadout = spacetimedb.reducer(
  { armor: t.u8(), shieldRecharge: t.u8(), color: t.string() },
  (ctx, { armor, shieldRecharge, color }) => {
    const p = ctx.db.player.identity.find(ctx.sender);
    if (!p) throw new SenderError('Player not found');
    if (armor < 1 || armor > 10) throw new SenderError('Invalid armor');
    if (shieldRecharge < 1 || shieldRecharge > 10) throw new SenderError('Invalid shieldRecharge');
    ctx.db.player.identity.update({ ...p, armor, shieldRecharge, color });
  }
);

// Join arena/match
export const joinArena = spacetimedb.reducer(
  { arenaId: t.string() },
  (ctx, { arenaId }) => {
    const p = ctx.db.player.identity.find(ctx.sender);
    if (!p) throw new SenderError('Player not found');
    ctx.db.player.identity.update({
      ...p,
      arenaId,
      alive: true,
      health: p.maxHealth,
      shield: p.maxShield,
      x: 0, y: 0, z: 0,
    });
  }
);
```

**Publish and generate bindings:**

```bash
# Terminal 1: Start local SpacetimeDB server
spacetime start

# Terminal 2: Publish the module
spacetime publish --server local --module-path spacetimedb rtank

# Generate TypeScript client bindings
spacetime generate --lang typescript \
  --out-dir src/module_bindings \
  --module-path spacetimedb
```

Regenerate bindings after every schema change. Never edit `src/module_bindings/` by hand.

### 1.2 Client Connection Setup — `src/main.tsx`

```tsx
// src/main.tsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import { SpacetimeDBProvider } from 'spacetimedb/react';
import { DbConnection, type ErrorContext } from './module_bindings';
import { App } from './App';

const HOST = import.meta.env.VITE_SPACETIMEDB_HOST ?? 'ws://localhost:3000';
const DB_NAME = import.meta.env.VITE_SPACETIMEDB_DB_NAME ?? 'rtank';
const TOKEN_KEY = `${HOST}/${DB_NAME}/auth_token`;

const connectionBuilder = DbConnection.builder()
  .withUri(HOST)
  .withDatabaseName(DB_NAME)
  .withToken(localStorage.getItem(TOKEN_KEY) || undefined)
  .onConnect((_conn, _identity, token) => {
    localStorage.setItem(TOKEN_KEY, token);
    console.log('[SpacetimeDB] Connected:', _identity.toHexString());
  })
  .onConnectError((_ctx: ErrorContext, err: Error) => {
    console.error('[SpacetimeDB] Connection error:', err);
  });

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <SpacetimeDBProvider connectionBuilder={connectionBuilder}>
      <App />
    </SpacetimeDBProvider>
  </StrictMode>
);
```

Add environment variables:

```bash
# .env.development
VITE_SPACETIMEDB_HOST=ws://localhost:3000
VITE_SPACETIMEDB_DB_NAME=rtank

# .env.production
VITE_SPACETIMEDB_HOST=wss://maincloud.spacetimedb.com
VITE_SPACETIMEDB_DB_NAME=rtank
```

### 1.3 SpacetimeDB ↔ Zustand Bridge — `src/game/SpacetimeSync.ts`

This is the critical integration layer. SpacetimeDB callbacks write into Zustand imperatively (matching the existing `useGameStore.getState()` pattern). The R3F `useFrame` loop reads from Zustand — no re-render path for positional data.

```typescript
// src/game/SpacetimeSync.ts
import { useEffect, useRef } from 'react';
import { useSpacetimeDB, useTable } from 'spacetimedb/react';
import { DbConnection, tables, reducers } from '../module_bindings';
import type { Player } from '../module_bindings/types';
import type { TankEntity, Vec3 } from '../utils/types';

// --- Remote player store (ref-based, no re-renders) ---
export interface RemotePlayerState {
  identity: string;
  x: number; y: number; z: number;
  rotation: number;
  turretRotation: number;
  velocityX: number; velocityZ: number;
  currentWeapon: string;
  isBoosting: boolean;
  isDrifting: boolean;
  vehicleMode: string;
  color: string;
  health: number;
  maxHealth: number;
  shield: number;
  maxShield: number;
  alive: boolean;
  arenaId: string;
  name: string;
}

// Global ref-based map — game loop reads this directly
export const remotePlayersRef: { current: Map<string, RemotePlayerState> } = {
  current: new Map()
};

// Local identity ref
export const localIdentityRef: { current: string | null } = { current: null };

/**
 * React component that bridges SpacetimeDB → remotePlayersRef.
 * Mount inside <Canvas> or at the App root.
 * Uses onInsert/onUpdate/onDelete callbacks to avoid re-render storms.
 */
export function SpacetimePlayerSync() {
  const { identity, isActive } = useSpacetimeDB();

  useEffect(() => {
    if (identity) {
      localIdentityRef.current = identity.toHexString();
    }
  }, [identity]);

  useTable(tables.player, undefined, {
    onInsert: (player: Player) => {
      const id = player.identity.toHexString();
      if (id === localIdentityRef.current) return; // skip self
      remotePlayersRef.current.set(id, playerRowToState(player));
    },
    onUpdate: (_old: Player, player: Player) => {
      const id = player.identity.toHexString();
      if (id === localIdentityRef.current) return;
      remotePlayersRef.current.set(id, playerRowToState(player));
    },
    onDelete: (player: Player) => {
      remotePlayersRef.current.delete(player.identity.toHexString());
    },
  });

  return null;
}

function playerRowToState(p: Player): RemotePlayerState {
  return {
    identity: p.identity.toHexString(),
    x: p.x, y: p.y, z: p.z,
    rotation: p.rotation,
    turretRotation: p.turretRotation,
    velocityX: p.velocityX, velocityZ: p.velocityZ,
    currentWeapon: p.currentWeapon,
    isBoosting: p.isBoosting,
    isDrifting: p.isDrifting,
    vehicleMode: p.vehicleMode,
    color: p.color,
    health: p.health, maxHealth: p.maxHealth,
    shield: p.shield, maxShield: p.maxShield,
    alive: p.alive,
    arenaId: p.arenaId,
    name: p.name,
  };
}
```

### 1.4 Throttled Position Send — integrate into existing game loop

Add to `BattleTanksScreen.tsx` `GameLoop` component, after `store.updatePlayer(updatedPlayer)`:

```typescript
// Inside GameLoop's useFrame, after store.updatePlayer(updatedPlayer):
const sendTimerRef = useRef(0);

// ... inside useFrame:
sendTimerRef.current += dt;
if (sendTimerRef.current >= 0.05) { // 20 updates/sec
  sendTimerRef.current = 0;
  conn.reducers.updatePosition({
    x: updatedPlayer.position.x,
    y: updatedPlayer.position.y,
    z: updatedPlayer.position.z,
    rotation: updatedPlayer.rotation,
    turretRotation: updatedPlayer.turretRotation,
    velocityX: updatedPlayer.velocity.x,
    velocityZ: updatedPlayer.velocity.z,
    currentWeapon: updatedPlayer.currentWeapon,
    isBoosting: updatedPlayer.isBoosting,
    isDrifting: updatedPlayer.isDrifting,
    vehicleMode: store.vehicleMode,
  });
}
```

### 1.5 Send Loadout on Match Init

In `BattleTanksScreen.tsx` `useEffect` where `initGame` is called:

```typescript
// After initGame(arenaRef.current, loadout):
conn.reducers.setLoadout({
  armor: loadout.armor,
  shieldRecharge: loadout.shieldRecharge,
  color: loadout.color,
});
conn.reducers.joinArena({ arenaId: arenaRef.current.id });
```

### Phase 1 Checklist

- [ ] `spacetime publish` succeeds, module visible at `http://localhost:3000`
- [ ] `spacetime generate` produces `src/module_bindings/` with no TS errors
- [ ] `SpacetimeDBProvider` wraps `<App />` in `main.tsx`
- [ ] Connection established (token stored in `localStorage`)
- [ ] Local player row created on connect, deleted on disconnect
- [ ] Loadout sent to server on match init
- [ ] Position sent at 20 Hz from game loop
- [ ] Game plays identically to singleplayer (no visible changes)

---

## Phase 2: Presence — See Other Players

*Goal: Render all connected players as tank meshes. Multiplayer is now visible.*

### 2.1 Mount the Sync Component

Add `<SpacetimePlayerSync />` inside both `BattleTanksScreen` and `OpenWorldScreen` Canvas:

```tsx
// BattleTanksScreen.tsx — inside <Canvas>
<SpacetimePlayerSync />
```

### 2.2 Remote Tank Renderer Component

```tsx
// src/components/3d/RemoteTanks.tsx
import { useRef } from 'react';
import { useFrame } from '@react-three/fiber';
import * as THREE from 'three';
import { remotePlayersRef, localIdentityRef } from '../../game/SpacetimeSync';
import { Tank } from './Tank';
import type { TankEntity, Vec3 } from '../../utils/types';

// Convert RemotePlayerState → partial TankEntity for the Tank renderer
function remoteToTankEntity(remote: RemotePlayerState, prevPos?: Vec3): TankEntity {
  return {
    id: `remote-${remote.identity}`,
    position: { x: remote.x, y: remote.y, z: remote.z },
    rotation: remote.rotation,
    turretRotation: remote.turretRotation,
    velocity: { x: remote.velocityX, y: 0, z: remote.velocityZ },
    health: remote.health,
    maxHealth: remote.maxHealth,
    shield: remote.shield,
    maxShield: remote.maxShield,
    shieldRechargeRate: 0,
    armor: 0,
    speed: Math.sqrt(remote.velocityX ** 2 + remote.velocityZ ** 2),
    isPlayer: false,
    color: remote.color,
    currentWeapon: remote.currentWeapon as any,
    boostCooldownRemaining: 0,
    isBoosting: remote.isBoosting,
    boostTimeRemaining: 0,
    isDrifting: remote.isDrifting,
    driftAngle: 0,
    driftSpeedFloor: 0,
    weaponCooldowns: { precision: 0, devastator: 0, suppressor: 0, shredder: 0, seeker: 0 },
    overheatLevel: 0,
    isOverheated: false,
    damageBuff: 1, damageBuffTimeRemaining: 0,
    speedBuff: 1, speedBuffTimeRemaining: 0,
    slowDebuff: 1, slowDebuffTimeRemaining: 0,
    loadoutSpeedMult: 1, loadoutDamageMult: 1,
    currentTargetId: null,
    invulnerableUntil: 0,
  };
}
```

Use a `useFrame`-driven approach that reads `remotePlayersRef` each frame and lerps positions for smooth interpolation:

```tsx
export function RemoteTanks({ arenaId }: { arenaId: string }) {
  const tanksRef = useRef<Map<string, TankEntity>>(new Map());
  const [, forceRender] = useState(0);
  const renderTickRef = useRef(0);

  useFrame((_, delta) => {
    const remotes = remotePlayersRef.current;
    let changed = false;

    // Remove stale
    for (const [id] of tanksRef.current) {
      if (!remotes.has(id)) {
        tanksRef.current.delete(id);
        changed = true;
      }
    }

    // Update / add
    for (const [id, remote] of remotes) {
      if (remote.arenaId !== arenaId || !remote.alive) continue;
      const existing = tanksRef.current.get(id);
      const entity = remoteToTankEntity(remote, existing?.position);

      // Lerp position for smoothness (interpolate between network updates)
      if (existing) {
        entity.position.x = existing.position.x + (entity.position.x - existing.position.x) * Math.min(1, delta * 15);
        entity.position.y = existing.position.y + (entity.position.y - existing.position.y) * Math.min(1, delta * 15);
        entity.position.z = existing.position.z + (entity.position.z - existing.position.z) * Math.min(1, delta * 15);
      }

      tanksRef.current.set(id, entity);
      if (!existing) changed = true;
    }

    // Re-render component tree only when players join/leave (not every frame)
    renderTickRef.current += delta;
    if (changed && renderTickRef.current > 0.1) {
      renderTickRef.current = 0;
      forceRender(n => n + 1);
    }
  });

  return (
    <>
      {[...tanksRef.current.values()].map(entity => (
        <Tank key={entity.id} entity={entity} showBoostTrail={entity.isBoosting} />
      ))}
    </>
  );
}
```

### 2.3 Add to Arena Scene

```tsx
// BattleTanksScreen.tsx — inside <Canvas>, after <EnemyTanks />
<RemoteTanks arenaId={arenaRef.current.id} />
```

### 2.4 Player Name Tags

```tsx
// src/components/3d/PlayerNameTag.tsx
import { Html } from '@react-three/drei';
import { remotePlayersRef } from '../../game/SpacetimeSync';

export function PlayerNameTag({ identity, position }: { identity: string; position: [number, number, number] }) {
  const remote = remotePlayersRef.current.get(identity);
  if (!remote) return null;

  return (
    <Html position={[position[0], position[1] + 4, position[2]]} center>
      <div style={{
        color: remote.color,
        fontFamily: "'Courier New', monospace",
        fontSize: '12px',
        textShadow: `0 0 8px ${remote.color}`,
        whiteSpace: 'nowrap',
        pointerEvents: 'none',
      }}>
        {remote.name}
      </div>
    </Html>
  );
}
```

### 2.5 Throttle Outgoing Reducer Calls

Already addressed in Phase 1.4 — the `sendTimerRef` ensures exactly 20 updates/sec max. The critical rule: **never call reducers inside `useFrame` without a time guard**.

### Phase 2 Checklist

- [ ] `<SpacetimePlayerSync />` mounted in both arena and open world screens
- [ ] Remote player tanks render with correct color, position, rotation
- [ ] Position interpolation is smooth (lerp factor ~15/sec)
- [ ] Players join/leave without mesh leaks (Map cleanup on `onDelete`)
- [ ] Reducer calls throttled to 20/sec
- [ ] No React re-render storms — position data flows through refs, not state
- [ ] Name tags visible above remote players

---

## Phase 3: Game Systems — Combat, AI, Weapons, Match Flow

*Goal: Full multiplayer combat — PvP damage, shared AI enemies, match lifecycle.*

### 3.1 PvP Combat — Damage System

**Server — new tables and reducers:**

```typescript
// spacetimedb/src/index.ts — add to schema

const damageLog = table(
  { name: 'damage_log', public: true },
  {
    id: t.u64().primaryKey().autoInc(),
    matchId: t.u32(),
    targetIdentity: t.identity(),
    sourceIdentity: t.identity(),
    amount: t.f32(),
    weaponType: t.string(),
    isCritical: t.bool(),
    timestamp: t.u64(),
  }
);

const killFeed = table(
  { name: 'kill_feed', public: true },
  {
    id: t.u64().primaryKey().autoInc(),
    matchId: t.u32(),
    killerIdentity: t.identity(),
    victimIdentity: t.identity(),
    weaponType: t.string(),
    timestamp: t.u64(),
  }
);
```

**Server — fire weapon reducer (server-validated):**

```typescript
export const fireWeapon = spacetimedb.reducer(
  {
    weaponType: t.string(),
    targetIdentity: t.identity(),
    shooterX: t.f32(), shooterY: t.f32(), shooterZ: t.f32(),
    shooterRotation: t.f32(),
  },
  (ctx, { weaponType, targetIdentity, shooterX, shooterY, shooterZ, shooterRotation }) => {
    const shooter = ctx.db.player.identity.find(ctx.sender);
    if (!shooter || !shooter.alive) return;

    const target = ctx.db.player.identity.find(targetIdentity);
    if (!target || !target.alive) return;

    // Validate weapon exists
    const WEAPON_STATS: Record<string, { damage: number; maxRange: number; optimalRange: number }> = {
      precision:   { damage: 25, maxRange: 100, optimalRange: 60 },
      devastator:  { damage: 50, maxRange: 55,  optimalRange: 35 },
      suppressor:  { damage: 10, maxRange: 45,  optimalRange: 25 },
      shredder:    { damage: 12, maxRange: 25,  optimalRange: 12 },
      seeker:      { damage: 90, maxRange: 120, optimalRange: 80 },
    };
    const wep = WEAPON_STATS[weaponType];
    if (!wep) throw new SenderError('Invalid weapon');

    // Server-side range validation
    const dx = shooterX - target.x;
    const dz = shooterZ - target.z;
    const dist = Math.sqrt(dx * dx + dz * dz);
    if (dist > wep.maxRange * 1.1) return; // 10% tolerance for latency

    // Calculate damage with falloff
    let damage = wep.damage;
    if (dist > wep.optimalRange) {
      const falloff = (dist - wep.optimalRange) / (wep.maxRange - wep.optimalRange);
      damage *= Math.max(0, 1 - falloff);
    }

    // Armor reduction (5% per point, floor 25%)
    const armorReduction = Math.max(0.25, 1 - target.armor * 0.05);
    damage *= armorReduction;

    // Apply to shield then health
    let shield = target.shield;
    let health = target.health;
    let remaining = damage;

    if (shield > 0) {
      const absorbed = Math.min(shield, remaining);
      shield -= absorbed;
      remaining -= absorbed;
    }
    health = Math.max(0, health - remaining);

    const alive = health > 0;
    ctx.db.player.identity.update({ ...target, shield, health, alive });

    // Log damage
    ctx.db.damageLog.insert({
      matchId: 0, // TODO: match tracking
      targetIdentity: targetIdentity,
      sourceIdentity: ctx.sender,
      amount: damage,
      weaponType,
      isCritical: false,
      timestamp: ctx.timestamp,
    });

    // Kill feed
    if (!alive) {
      ctx.db.killFeed.insert({
        matchId: 0,
        killerIdentity: ctx.sender,
        victimIdentity: targetIdentity,
        weaponType,
        timestamp: ctx.timestamp,
      });
      // Award score
      const killer = ctx.db.player.identity.find(ctx.sender);
      if (killer) {
        ctx.db.player.identity.update({ ...killer, score: killer.score + 25 });
      }
    }
  }
);
```

**Client — fire at remote player:**

In the game loop, when the player fires and the target is a remote player (identity starts with `remote-`), call the server reducer instead of local `fireHitscan`:

```typescript
// In GameLoop useFrame, where player fire is processed:
if (fireTarget && isRemotePlayer(fireTarget.id)) {
  // PvP: send to server for validation
  const targetIdentity = fireTarget.id.replace('remote-', '');
  conn.reducers.fireWeapon({
    weaponType: updatedPlayer.currentWeapon,
    targetIdentity: Identity.fromHexString(targetIdentity),
    shooterX: updatedPlayer.position.x,
    shooterY: updatedPlayer.position.y,
    shooterZ: updatedPlayer.position.z,
    shooterRotation: updatedPlayer.rotation,
  });
  // Optimistic: play muzzle flash + tracer locally
  // Server confirms damage via player table update
} else {
  // PvE: existing local hitscan against AI enemies
  // (until Phase 3.2 moves AI server-side)
}
```

### 3.2 AI Enemies — ZeroBytes-Aware Design

The ZeroBytes architecture creates two distinct multiplayer modes for AI enemies:

#### Open World: Deterministic Spawns + Kill-State Sync (No Server AI)

In the open world, enemy spawn positions are deterministic from `generateValleyEnemySpawns(valleyX, valleyZ, count, worldSeed)`. Every client independently generates identical enemy positions. The server does **not** need to run AI — each client runs AI locally against its own player. What needs syncing:

- **`enemyKill` table** — when any player kills an enemy, insert its ID. All clients check this table to filter out dead enemies.
- **`worldConfig.gameTime`** — drives `resolveValleyCombatState()` which determines how many enemies survived NPC defender combat. All clients compute this identically from shared `gameTime`.

```typescript
// Server reducer: player kills an open-world enemy
export const killEnemy = spacetimedb.reducer(
  { enemyId: t.string(), valleyId: t.string() },
  (ctx, { enemyId, valleyId }) => {
    const shooter = ctx.db.player.identity.find(ctx.sender);
    if (!shooter || !shooter.alive) return;

    // Prevent double-kill
    const existing = ctx.db.enemyKill.id.find(enemyId);
    if (existing) return;

    ctx.db.enemyKill.insert({
      id: enemyId,
      killerIdentity: ctx.sender,
      valleyId,
      killedAt: ctx.timestamp,
    });

    // Award score
    ctx.db.player.identity.update({ ...shooter, score: shooter.score + 25 });
  }
);
```

**Client — open world enemy filtering:**

```typescript
// On valley enter, generate enemies deterministically (existing code unchanged)
const valley = generateValley(valleyX, valleyZ, worldSeed);
const enemies = valley.enemySpawns.map((spawn, i) => createValleyEnemy(spawn, i, valley));

// Filter out enemies already killed by any player (from enemyKill table)
const killedIds = new Set([...conn.db.enemyKill.iter()].map(k => k.id));
const aliveEnemies = enemies.filter(e => !killedIds.has(e.id));
```

This approach means:
- **Zero AI state in SpacetimeDB** — no `aiEnemy` table, no server tick, no 10 Hz reducer cost
- **O(1) enemy reconstruction** — any client joining mid-session generates enemies from position + filters by kill set
- **CombatLayer compatibility** — `resolveValleyCombatState(valley, worldSeed, gameTime, playerKills)` works identically since `playerKills` = count of `enemyKill` rows for that valley

#### Arena Mode: Server-Authoritative AI (PvP Context)

In arena mode (closed maps with fixed enemy counts), server-authoritative AI is needed because multiple players fight the same enemies simultaneously. Here the original `aiEnemy` table approach applies:

```typescript
const aiEnemy = table(
  { name: 'ai_enemy', public: true },
  {
    id: t.string().primaryKey(),
    arenaId: t.string(),
    x: t.f32(), y: t.f32(), z: t.f32(),
    rotation: t.f32(),
    turretRotation: t.f32(),
    health: t.f32(),
    maxHealth: t.f32(),
    shield: t.f32(),
    aiState: t.string(),
    currentWeapon: t.string(),
    color: t.string(),
    alive: t.bool(),
  }
);
```

Server runs the AI state machine at 10 Hz (scheduled reducer). Arena enemy spawns come from the `ArenaDefinition.enemySpawns` array, which is also deterministic (hard-coded in `data/arenas.ts`).

**Client migration path:**
1. Open world: keep local AI as-is, add `enemyKill` table filtering
2. Arena mode: when `conn.isActive`, subscribe to `aiEnemy` table, render from it, fire calls go to `fireAtAI` reducer

### 3.3 Loot & Secrets — ZeroBytes Delta Sync

Loot items and secrets are fully deterministic from position. The server never stores item definitions — only which ones have been collected.

**Server — collect loot reducer:**

```typescript
export const collectLoot = spacetimedb.reducer(
  { lootId: t.string() },
  (ctx, { lootId }) => {
    const player = ctx.db.player.identity.find(ctx.sender);
    if (!player || !player.alive) return;

    // Prevent double-collect
    const existing = ctx.db.collectedLoot.id.find(lootId);
    if (existing) return; // another player already got it

    ctx.db.collectedLoot.insert({
      id: lootId,
      collectorIdentity: ctx.sender,
      collectedAt: ctx.timestamp,
    });

    // Score bonus (item value is deterministic from position — server can re-derive)
    // Or just award a flat amount since the exact item is ZeroBytes-computable
    ctx.db.player.identity.update({ ...player, score: player.score + 10 });
  }
);

export const discoverSecret = spacetimedb.reducer(
  { secretId: t.string() },
  (ctx, { secretId }) => {
    const player = ctx.db.player.identity.find(ctx.sender);
    if (!player || !player.alive) return;

    const existing = ctx.db.discoveredSecret.id.find(secretId);
    if (existing) return;

    ctx.db.discoveredSecret.insert({
      id: secretId,
      discovererIdentity: ctx.sender,
      discoveredAt: ctx.timestamp,
    });
  }
);
```

**Client — filter collected loot on valley enter:**

```typescript
// Generate loot deterministically (existing code)
const lootSpawns = generateValleyLoot(valleyX, valleyZ, worldSeed, valley.treasureValue, valley.type);

// Filter out already-collected items (from collectedLoot table)
const collectedIds = new Set([...conn.db.collectedLoot.iter()].map(l => l.id));
const availableLoot = lootSpawns.filter(l => !collectedIds.has(l.id));
```

The same pattern applies to secrets — generate from `generateValleySecrets`, filter by `discoveredSecret` table. This replaces the `localStorage` key `rt-secret-progress` with server-persisted data visible to all players (first-to-find gets credit).

### 3.4 Linger Zones (Devastator DOT)

**Server table (PvP only — linger zones in PvE stay local):**

```typescript
const lingerZone = table(
  { name: 'linger_zone', public: true },
  {
    id: t.string().primaryKey(),
    arenaId: t.string(),
    x: t.f32(), z: t.f32(),
    radius: t.f32(),
    damage: t.f32(),
    ownerIdentity: t.identity(),
    timeRemaining: t.f32(),
    color: t.string(),
  }
);
```

Linger zones are created by the server when a devastator fires in PvP. A scheduled reducer ticks them down and applies DOT damage to any player in radius. In PvE open world, linger zones remain local (no cross-player impact).

### 3.5 Nemesis System — Delta Sync

The nemesis system uses `coordsToSeed` for roaming and `mulberry32` for name generation. Roaming positions are fully deterministic from `(nemesisId, gameTurn, worldSeed)`. The server stores only the mutable nemesis records:

```typescript
const nemesisRecord = table(
  { name: 'nemesis_record', public: true },
  {
    id: t.string().primaryKey(),
    name: t.string(),
    rank: t.string(),
    wins: t.u32(),
    xp: t.u32(),
    level: t.u8(),
    baseHealth: t.f32(),
    baseDamage: t.f32(),
    baseSpeed: t.f32(),
    lastEncounterTurn: t.u32(),
    dialogueIndex: t.u8(),
    defeated: t.bool(),
    defeatedTurn: t.u32(),    // 0 if not defeated
    createdTurn: t.u32(),
    entourageSize: t.u8(),
  }
);
```

Roaming position is **not stored** — it's recomputed by each client from `coordsToSeed(hashStringToInt(record.id), gameTurn, 0, worldSeed + 800000)`. This means nemesis movement is automatically synchronized with zero bandwidth cost. The only reducers needed are `onPlayerKilled` (create/promote nemesis) and `onNemesisDefeated` (mark defeated).

### 3.6 Match Lifecycle

**Server — match management reducers:**

```typescript
export const createMatch = spacetimedb.reducer(
  { arenaId: t.string(), maxPlayers: t.u8() },
  (ctx, { arenaId, maxPlayers }) => {
    ctx.db.matchState.insert({
      arenaId,
      gameTime: 0,
      isPaused: false,
      isGameOver: false,
      isVictory: false,
      maxPlayers,
      createdAt: ctx.timestamp,
    });
  }
);

export const endMatch = spacetimedb.reducer(
  { matchId: t.u32() },
  (ctx, { matchId }) => {
    const match = ctx.db.matchState.id.find(matchId);
    if (!match) return;
    ctx.db.matchState.id.update({ ...match, isGameOver: true });
  }
);
```

**Client — subscribe to match state:**

```typescript
// Use useTable for UI-rate match state (scoreboard, game over screen)
const [matches] = useTable(tables.matchState);
const currentMatch = matches.find(m => m.arenaId === currentArenaId && !m.isGameOver);
```

### 3.5 Kill Feed UI

**Client — subscribe to kill feed for HUD display:**

```typescript
const [kills] = useTable(tables.killFeed);
// Show last 5 kills in HUD overlay
const recentKills = kills
  .sort((a, b) => Number(b.timestamp - a.timestamp))
  .slice(0, 5);
```

### Phase 3 Checklist

- [ ] PvP damage flows through `fireWeapon` reducer with server validation
- [ ] AI enemies run server-side, all clients render from `ai_enemy` table
- [ ] Linger zones created/ticked server-side, rendered by all clients
- [ ] Match create/join/end lifecycle works
- [ ] Kill feed displays in HUD
- [ ] Singleplayer still works when offline (fallback to local state)

---

## Phase 4: Authority & Validation

*Goal: Prevent cheating. All trust-sensitive logic runs server-side.*

### What Must Be Server-Authoritative

| System | Why | Validation |
|---|---|---|
| **Damage calculations** | Clients could send arbitrary damage | Server computes damage from weapon stats + range + armor |
| **Health/shield state** | Clients could set health to max | Only server mutates `health`/`shield` columns |
| **Weapon cooldowns** | Clients could fire at infinite rate | Server tracks last-fire timestamp per weapon per player |
| **Movement speed** | Clients could teleport | Server validates position delta vs max speed × elapsed time |
| **Score** | Clients could inflate score | Server awards score only in kill/loot reducers |
| **Loot collection** | Clients could claim remote loot | Server validates proximity before granting item |

### Server-Side Weapon Cooldown Tracking

```typescript
// Add to player table:
// lastFireTimestamps: JSON string of { [weapon]: timestamp }

export const fireWeapon = spacetimedb.reducer(
  { /* ... */ },
  (ctx, args) => {
    const shooter = ctx.db.player.identity.find(ctx.sender);
    // ...

    // Cooldown check
    const cooldowns: Record<string, number> = {
      precision: 333, devastator: 1000, suppressor: 125,
      shredder: 667, seeker: 8000,
    };
    const minInterval = cooldowns[args.weaponType] ?? 1000;
    // Use ctx.timestamp (milliseconds) to enforce
    // Compare against stored lastFireTime for this weapon
    // Reject if too soon
  }
);
```

### Position Validation

```typescript
export const updatePosition = spacetimedb.reducer(
  { /* ... */ },
  (ctx, args) => {
    const p = ctx.db.player.identity.find(ctx.sender);
    if (!p) throw new SenderError('Player not found');

    // Validate position delta — max speed is ~32 u/s (boosted forward)
    // With 50ms send interval, max delta per update = 32 * 0.06 = 1.92 units
    const dx = args.x - p.x;
    const dz = args.z - p.z;
    const dist = Math.sqrt(dx * dx + dz * dz);
    const MAX_DELTA = 4.0; // generous tolerance for latency spikes
    if (dist > MAX_DELTA) {
      // Clamp to max allowed movement in the player's direction
      const scale = MAX_DELTA / dist;
      args.x = p.x + dx * scale;
      args.z = p.z + dz * scale;
    }

    ctx.db.player.identity.update({ ...p, ...args });
  }
);
```

### Phase 4 Checklist

- [x] Damage computed server-side from weapon stats (fireWeapon reducer)
- [x] Weapon cooldowns enforced server-side (lastFireTime + per-weapon cooldownMs)
- [x] Position deltas validated (no teleportation) (updatePosition clamps to MAX_DELTA=4.0)
- [x] Score only awarded through server reducers (killEnemy +25, collectLoot +10, fireWeapon kill +25)
- [x] Health/shield only mutated by server (fireWeapon applies damage, updatePosition regens shield)
- [x] Shield regeneration server-side (shieldRecharge stat, ticked in updatePosition)
- [x] Self-fire prevention (fireWeapon rejects ctx.sender === targetIdentity)
- [x] Range validation uses server-known position (not client-claimed shooterX/Z)

---

## Phase 5: Auth & Identity

*Goal: Named players, persistent identity across sessions, progression.*

### 5.1 Player Name Setting

```typescript
// Server reducer
export const setPlayerName = spacetimedb.reducer(
  { name: t.string() },
  (ctx, { name }) => {
    const trimmed = name.trim();
    if (trimmed.length < 1 || trimmed.length > 16) throw new SenderError('Name must be 1-16 characters');
    // Sanitize: alphanumeric + dashes + underscores only
    if (!/^[a-zA-Z0-9_-]+$/.test(trimmed)) throw new SenderError('Invalid characters in name');

    const p = ctx.db.player.identity.find(ctx.sender);
    if (!p) throw new SenderError('Player not found');
    ctx.db.player.identity.update({ ...p, name: trimmed });
  }
);
```

**Client — name entry on first connect:**

```tsx
const setName = useReducer(reducers.setPlayerName);
// Show name prompt if player name is default 'TANK-XXXXXX'
setName({ name: userInput });
```

### 5.2 Persistent Progression Table

```typescript
const playerProgress = table(
  { name: 'player_progress', public: false }, // private — only owner sees
  {
    identity: t.identity().primaryKey(),
    totalKills: t.u32(),
    totalDeaths: t.u32(),
    totalPlaytime: t.u64(),     // seconds
    totalScore: t.u64(),
    matchesPlayed: t.u32(),
    matchesWon: t.u32(),
    discoveredSecrets: t.string(), // JSON array of secret IDs
    // Inventory serialized as JSON (complex nested structure)
    inventoryJson: t.string(),
  }
);
```

This replaces `localStorage` keys `rt-progress`, `rt-secret-progress`, and `rt-inventory` with server-persisted data.

### 5.3 Token-Based Identity

Already established in Phase 1 via `localStorage` token keyed by `HOST/DB_NAME`. This gives:
- **Same browser, same device:** Automatic reconnection with same identity
- **Cross-device:** Use SpacetimeAuth OAuth flow (Google/Discord/GitHub)

For cross-device auth: https://spacetimedb.com/docs/spacetimeauth/react-integration

### Phase 5 Checklist

- [x] Players can set custom names (validated server-side, persisted in playerProgress.displayName)
- [x] Progression persisted in SpacetimeDB (playerProgress table + syncProgress reducer)
- [x] Token persisted in localStorage for session continuity (Phase 1)
- [x] Name tags show custom names above remote players (Phase 2, uses player.name)
- [x] Callsign UI in Settings screen (1-16 chars, alphanumeric + dash/underscore)
- [x] Progress syncs on screen unmount (BattleTanks + OpenWorld) and manual save
- [x] Display name restored on reconnect (onConnect reads playerProgress.displayName)

---

## File Change Summary

| File | Action | Phase |
|---|---|---|
| `spacetimedb/src/index.ts` | **Create** — full server module | 1 |
| `src/main.tsx` | **Modify** — wrap with `SpacetimeDBProvider` | 1 |
| `src/game/SpacetimeSync.ts` | **Create** — bridge component + remote player store | 1 |
| `src/components/3d/RemoteTanks.tsx` | **Create** — render remote player meshes | 2 |
| `src/components/3d/PlayerNameTag.tsx` | **Create** — name tags above remote players | 2 |
| `src/screens/BattleTanksScreen.tsx` | **Modify** — add `<SpacetimePlayerSync>`, `<RemoteTanks>`, throttled send, PvP fire | 1–3 |
| `src/screens/OpenWorldScreen.tsx` | **Modify** — same as above for open world | 1–3 |
| `src/components/ui/KillFeed.tsx` | **Create** — kill feed HUD overlay | 3 |
| `src/components/ui/HUD.tsx` | **Modify** — add kill feed, player count | 3 |
| `src/game/GameState.ts` | **Modify** — add connection state, fallback offline mode | 1 |
| `.env.development` | **Create** — SpacetimeDB host/db vars | 1 |
| `.env.production` | **Create** — production SpacetimeDB vars | 1 |
| `package.json` | **Modify** — add `spacetimedb` dependency | 1 |

---

## Deployment

```bash
# Local development
spacetime start                                    # Terminal 1
spacetime publish --server local --module-path spacetimedb rtank  # Terminal 2
npm run dev                                        # Terminal 3

# Production (Maincloud free tier — 1.5M reducer calls/month)
spacetime publish --server maincloud --module-path spacetimedb rtank
```

Budget estimate at 20 updates/sec per player:
- 1 player = 72,000 calls/hour
- 4 concurrent players = 288,000 calls/hour
- Free tier supports ~5 hours of 4-player sessions/month
- Consider reducing to 10 updates/sec for production (still smooth with interpolation)

---

## Risk & Gotchas

| Risk | Mitigation |
|---|---|
| **R3F re-render storms from `useTable`** | Use `onInsert`/`onUpdate`/`onDelete` callbacks → write to `remotePlayersRef` (ref-based Map). Game loop reads ref directly. Only UI-rate data (kill feed, scoreboard) uses reactive `useTable`. |
| **Zustand + SpacetimeDB dual-source-of-truth** | Clear ownership: local player physics stays in Zustand (optimistic). SpacetimeDB is source of truth for remote players, damage, and scores. Local state reconciles on server updates. |
| **AI state machine in arena mode** | Only arena mode needs server-side AI (10 Hz scheduled reducer). Open world AI stays client-local — spawns are deterministic, only kills are synced. |
| **74-field TankEntity too large to network** | Only sync 12 fields (position, rotation, turret, velocity, weapon, boost, drift, vehicle mode, color, health, shield, alive). Everything else is derived or local. |
| **Hitscan weapons are instant** | Server validates range + cooldown. Client shows tracer optimistically. If server rejects (range hack), damage doesn't apply — no rollback needed since tracers are cosmetic. |
| **Open world valley streaming** | Players in different valleys don't need to see each other. Use `arenaId` field to filter. Subscribe only to players in current valley. |
| **ZeroBytes worldSeed must match across all clients** | The `worldConfig` table stores the canonical `worldSeed` (u32). Clients read it on connect and pass it to all generation functions. If a client has a mismatched seed, the entire world is wrong. |
| **ZeroBytes floating-point determinism** | `mulberry32` and `coordsToSeed` use integer math (`Math.imul`, `>>> 0`) which is deterministic across all JS engines. `Math.sin`/`Math.cos` in simplex noise could theoretically differ by ULP across platforms — test on target browsers. |
| **ZeroBytes kill-state consistency** | Two players killing the same enemy simultaneously: `enemyKill` insert uses enemy ID as primary key, so only one insert succeeds. The other client sees the kill via subscription and removes the enemy locally. No double-count. |
| **ZeroBytes loot race condition** | Two players reaching the same loot: `collectedLoot` insert by ID — first writer wins. Second player sees it collected via subscription. UI should optimistically hide loot on pickup attempt, then re-show if server rejects. |
| **Module schema changes** | Use incremental migrations: https://spacetimedb.com/docs/how-to/incremental-migrations |
| **`onInsert` fires for pre-existing rows** | All `onInsert` handlers in `SpacetimeSync.ts` are idempotent (Map.set overwrites). |
| **Reducer call budget on Maincloud** | Reduce position send rate to 10/sec in production. Use dead reckoning on client to fill gaps. |
| **Identity loss on clear browser data** | Token in `localStorage` keyed by host/db. Document that clearing site data resets identity. |

---

## Reference

- SpacetimeDB Docs: https://spacetimedb.com/docs
- TypeScript SDK: https://spacetimedb.com/docs/sdks/typescript
- TypeScript Module Reference: https://spacetimedb.com/docs/modules/typescript
- SpacetimeAuth React: https://spacetimedb.com/docs/spacetimeauth/react-integration
- Incremental Migrations: https://spacetimedb.com/docs/how-to/incremental-migrations
- GitHub: https://github.com/clockworklabs/SpacetimeDB
