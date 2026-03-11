# TANK Nemesis System — Implementation Plan

## Description

The TANK Nemesis System introduces persistent, procedurally-generated **Ace Tank Commanders** that roam the open world, grow stronger when the player is defeated, and create emergent rivalries across play sessions. Adapted from the `/nemesis-mecha-JSX/` component architecture (enemy tracking, rank progression, stat scaling, dialogue indexing) but fully re-themed for the TRON-inspired wireframe tank combat world of rTank-carmero.

**Core loop:** When the player is destroyed in combat, the enemy who landed the killing blow is promoted to (or strengthened as) a **Nemesis**. Nemeses persist in `localStorage`, roam between valleys, grow in rank/stats/entourage size, and can eventually **invade safe settlements** at the highest threat levels. They communicate via the existing dialog system using "hacked comms" — a fixed static-interference portrait with speaker name "UNKNOWN", delivering taunts, gloats, and monologues from a pool of 25 evil tank commander lines.

**Key design principles:**
- Zero new UI screens — nemesis taunts use the existing `DialogOverlay` toast system
- Zero new 3D models — nemeses are standard `<Tank>` components with a unique crimson color and optional glow
- Deterministic roaming via position-as-seed — nemesis valley placement is computable from nemesis ID + world seed + turn counter
- Permanent fields on `TankEntity` distinguish nemesis tanks from regular enemies at the AI level
- All nemesis state persisted in a single `localStorage` key (`rt-nemesis`)

---

## Functionality

### F1. Nemesis Record Data Model

Each nemesis is tracked as a `NemesisRecord` stored outside the zustand game state (in localStorage, loaded into the store on init):

```typescript
interface NemesisRecord {
  id: string;                    // unique: 'nemesis-{killSeed}'
  name: string;                  // procedurally generated (e.g. "COMMANDER VEXAR")
  rank: NemesisRank;             // progression tier
  wins: number;                  // times this nemesis has killed the player
  xp: number;                    // internal XP tracker (wins * 100 + bonus)
  level: number;                 // derived from xp thresholds
  baseHealth: number;            // starting health when first created
  baseDamage: number;            // starting damage multiplier
  baseSpeed: number;             // starting speed multiplier
  currentValleyX: number;        // which valley grid cell they roam in
  currentValleyZ: number;        // which valley grid cell they roam in
  lastEncounterTurn: number;     // game turn of last encounter with player
  dialogueIndex: number;         // 0-24 index into NEMESIS_MONOLOGUES
  defeated: boolean;             // true if player has killed them
  defeatedTurn: number | null;   // turn when defeated (null if alive)
  createdTurn: number;           // turn when first promoted
  color: string;                 // tank color (always NEMESIS_COLOR)
  entourageSize: number;         // number of escort tanks (0 at low level, up to 4)
  canInvadeSafe: boolean;        // true at highest threat level
}
```

### F2. Nemesis Rank Progression

Five ranks with XP thresholds:

| Rank | XP Threshold | Level | Entourage | Can Invade Safe? | Stat Bonus |
|------|-------------|-------|-----------|-----------------|------------|
| `RAIDER` | 0 | 1 | 0 | No | +0% |
| `MARAUDER` | 100 | 2 | 1 | No | +15% |
| `WARLORD` | 250 | 3 | 2 | No | +30% |
| `OVERLORD` | 500 | 4 | 3 | No | +50% |
| `TYRANT` | 900 | 5 | 4 | **Yes** | +75% |

```typescript
type NemesisRank = 'RAIDER' | 'MARAUDER' | 'WARLORD' | 'OVERLORD' | 'TYRANT';

const NEMESIS_RANKS: NemesisRank[] = ['RAIDER', 'MARAUDER', 'WARLORD', 'OVERLORD', 'TYRANT'];
const NEMESIS_XP_THRESHOLDS = [0, 100, 250, 500, 900];
const NEMESIS_ENTOURAGE_SIZE = [0, 1, 2, 3, 4];
const NEMESIS_STAT_BONUS = [0, 0.15, 0.30, 0.50, 0.75];
```

XP is gained: `+100` per player kill, `+25` per game turn survived. Level/rank computed from thresholds (same pattern as the mecha-JSX `RANK_XP` system).

### F3. Nemesis Creation (On Player Death)

When `player.health <= 0` in `processDamageEvents()`:

1. Identify the **killer** — the enemy whose `DamageEvent.source` matches the lethal hit
2. Check if that enemy is already a nemesis (`enemy.isNemesis === true`) — if so, increment their `wins`, add `+100 XP`, recalculate rank/stats
3. If the killer is a regular enemy AND fewer than **4 active (non-defeated) nemeses** exist:
   - Create a new `NemesisRecord` from the killer's stats
   - Generate a name using a seed derived from the enemy ID and kill timestamp
   - Set initial rank to `RAIDER`, wins to `1`, xp to `100`
4. If 4 active nemeses already exist and killer is new — no new nemesis, but the highest-level existing nemesis gains `+50 XP` bonus ("absorbs the kill glory")
5. Trigger a "hacked comms" taunt dialog after a 1.5s delay (after the death explosion)

### F4. Nemesis Stat Scaling

When a nemesis is spawned as a `TankEntity` in a valley, their stats are computed from the base values plus rank bonus:

```typescript
function computeNemesisStats(record: NemesisRecord): Partial<TankEntity> {
  const bonus = NEMESIS_STAT_BONUS[record.level - 1];
  return {
    health: Math.round(record.baseHealth * (1 + bonus)),
    maxHealth: Math.round(record.baseHealth * (1 + bonus)),
    shield: Math.round(50 * (1 + bonus)),
    maxShield: Math.round(50 * (1 + bonus)),
    armor: Math.min(10, 5 + record.level),
    damageBuff: 1 + bonus,
    damageBuffTimeRemaining: Infinity,  // permanent — never decays
    speedBuff: 1 + bonus * 0.5,        // speed scales slower than damage
    speedBuffTimeRemaining: Infinity,   // permanent
  };
}
```

The `Infinity` time remaining ensures the existing buff-decay logic in `Physics.ts` lines 221-239 never resets these to 1. This is the simplest integration: **zero changes** to the timer decay code.

### F5. Nemesis Roaming (Deterministic Valley Placement)

Each game turn (valley transition), every living nemesis "moves" to a new valley. Their position is deterministic:

```typescript
function getNemesisValley(nemesisId: string, turn: number, worldSeed: number): { vx: number; vz: number } {
  const seed = coordsToSeed(
    hashStringToInt(nemesisId),
    turn,
    0,
    worldSeed + 800000
  );
  const rng = mulberry32(seed);
  // Roam within a radius that grows with their level
  const record = getNemesisById(nemesisId);
  const roamRadius = 2 + (record?.level ?? 1);
  const vx = Math.floor((rng() - 0.5) * 2 * roamRadius);
  const vz = Math.floor((rng() - 0.5) * 2 * roamRadius);
  return { vx, vz };
}
```

A `hashStringToInt` helper converts the nemesis ID string to a numeric seed. The roam radius grows with level: level 1 nemeses stay within 3 valleys of origin, level 5 can appear anywhere within 7 valleys — including `(0,0)` home base if `canInvadeSafe` is true.

### F6. Nemesis Spawning in Valleys

When the player enters a valley (the existing `enterNewValley` block in `useFrame`), after spawning regular enemies:

1. Check all active nemesis records: does any nemesis's `currentValleyX/Z` match this valley?
2. If yes, spawn the nemesis as an additional `TankEntity` with:
   - `isNemesis: true` flag
   - Stats from `computeNemesisStats()`
   - Color: `NEMESIS_COLOR` (crimson `'#FF2222'`)
   - AI state: `'aggressive_chase'` (immediately hunts player)
   - Unique ID: the nemesis record's `id`
3. Also spawn their entourage as regular enemies with slightly boosted stats
4. Trigger a "hacked comms" taunt dialog: `"[NEMESIS NAME] has been detected in this sector!"`

### F7. Nemesis AI Behavior

Nemesis tanks use the existing `updateEnemyAI()` function but with enhanced parameters. The AI system checks `enemy.isNemesis` to apply overrides:

- **Detection range:** `AI_DEFAULTS.detectionRange * 1.5` (120 units vs 80)
- **Never flees:** Skip the `retreatThreshold` state transition — nemeses fight to the death
- **Weapon selection:** Nemeses always pick the optimal weapon for range (same `chooseWeapon()` but at rank 3+ they also use `'seeker'` at long range and `'devastator'` for area control)
- **Move speed:** `AI_DEFAULTS.moveSpeed * (1 + NEMESIS_STAT_BONUS[level-1] * 0.5)`

Implementation: a single `if (enemy.isNemesis)` guard block at the top of `updateEnemyAI()` that overrides `detectionRange`, `retreatThreshold`, and `moveSpeed` before the existing FSM runs.

### F8. Nemesis Defeat

When a nemesis tank is destroyed (`health <= 0`):

1. Mark the `NemesisRecord` as `defeated: true`, set `defeatedTurn`
2. Award **bonus score:** `SCORE_PER_KILL * record.level * 2`
3. Trigger a defeat monologue via "hacked comms" (the nemesis's `dialogueIndex` selects from `NEMESIS_MONOLOGUES`)
4. Play the `'LARGE'` explosion effect (same as player death — nemeses go out with a bang)
5. Persist updated records to localStorage

### F9. Hacked Comms Dialog System

Nemesis communications use the existing `DialogOverlay` toast system. The "hacked comms" aesthetic:

- **Portrait:** A fixed static image at path `${BASE}npc/chat/nemesis_chat.png` — the same image for all nemeses (representing jammed/corrupted comms). If this image doesn't exist, the system falls back to `ProceduralPortrait` with `npcType = 'nemesis'` (a red-tinted wireframe face).
- **Speaker name:** Uses `'UNKNOWN'` for the name tag (or the nemesis name if they've been encountered 2+ times)
- **Border color:** Crimson (`#FF2222`) instead of the normal NPC colors
- **Text:** Selected from `NEMESIS_MONOLOGUES` pool

Integration: Add `'nemesis'` to the `NPCType` union in `npcDialogs.ts`, add a `NEMESIS_COLOR` entry to `NPC_TYPE_COLORS`, and create nemesis dialog trees on-the-fly using a factory function.

### F10. Nemesis Monologues (25 Lines)

The monologue pool is split into categories, selected by context:

**Taunts (on spawn/encounter, indices 0-7):**
```
0: "Your signal is weak, commander. Just like your armor."
1: "I've been tracking you across three sectors. Nowhere left to run."
2: "Every valley you enter, I'm already there. Waiting."
3: "My targeting systems have your signature memorized."
4: "You think these mountains will save you? I own this grid."
5: "Last time was just a warm-up. This time, I finish it."
6: "Your wingman can't save you from what's coming."
7: "I've upgraded since our last dance. Have you?"
```

**Gloats (on killing the player, indices 8-15):**
```
8:  "Another notch on my barrel. You never learn."
9:  "Predictable. Your tactics haven't changed since our first meeting."
10: "That's {wins} times now. Are you even trying?"
11: "Your wreckage makes excellent cover. Thanks for that."
12: "Uploading your destruction to the grid. Everyone will see."
13: "They'll need to send someone better. You're not enough."
14: "I almost felt that last shot. Almost."
15: "Stay down this time. The grid has no place for the weak."
```

**Defeat monologues (on nemesis death, indices 16-24):**
```
16: "Impossible... my systems were... superior..."
17: "This changes nothing. The grid will send another."
18: "You got lucky. Next time there won't be luck."
19: "My data... will be... uploaded... someone will finish... this..."
20: "So this is how a machine dies. Cold. In the dark."
21: "Fine. You've earned this sector. For now."
22: "I underestimated you, commander. I won't make that mistake in my next cycle."
23: "The grid... remembers... everything..."
24: "Tell them... tell them TYRANT fell fighting. Not running. Never running."
```

Note: Line 10 uses `{wins}` as a template variable replaced at runtime with the nemesis's actual win count.

### F11. Safe Zone Invasion

At `TYRANT` rank (level 5, XP >= 900), a nemesis gains `canInvadeSafe: true`. This means:

- Their roaming algorithm **no longer excludes valley (0,0)** (home base)
- Their roaming algorithm **no longer excludes settlement-type valleys**
- When spawned in a settlement valley, they and their entourage appear as hostile tanks alongside friendly NPCs
- A special warning dialog fires: `"WARNING: HOSTILE ACE DETECTED IN SAFE ZONE"`
- Settlement NPCs are NOT targetable by the nemesis (nemesis only hunts the player)

### F12. Persistence

All nemesis records are stored in localStorage under key `rt-nemesis`:

```typescript
interface NemesisState {
  records: NemesisRecord[];
  gameTurn: number;           // incremented on each valley transition
  totalNemesesCreated: number; // running counter for unique ID generation
}
```

Loaded on game init via the existing `loadJSON()` helper. Saved after every nemesis-related state change (creation, promotion, defeat, roaming update).

### F13. HUD Integration

A minimal HUD indicator when a nemesis is present in the current valley:

- **Position:** Top-center, below the existing score display
- **Content:** Pulsing red text: `"ACE DETECTED: [RANK] [NAME]"` with health bar
- **Visibility:** Only when `enemies.some(e => e.isNemesis && e.health > 0)`
- **Style:** Crimson text with `textShadow: '0 0 15px #FF222280'`, `animation: pulse 1.5s infinite`

---

## Technical Implementation

### Step 1: Add Nemesis Types and Constants

**File: `src/utils/types.ts`**

Add the `NemesisRecord`, `NemesisRank`, and `NemesisState` interfaces after the existing `PlayerProgress` interface:

```typescript
// --- Nemesis System Types ---

export type NemesisRank = 'RAIDER' | 'MARAUDER' | 'WARLORD' | 'OVERLORD' | 'TYRANT';

export interface NemesisRecord {
  id: string;
  name: string;
  rank: NemesisRank;
  wins: number;
  xp: number;
  level: number;
  baseHealth: number;
  baseDamage: number;
  baseSpeed: number;
  currentValleyX: number;
  currentValleyZ: number;
  lastEncounterTurn: number;
  dialogueIndex: number;
  defeated: boolean;
  defeatedTurn: number | null;
  createdTurn: number;
  color: string;
  entourageSize: number;
  canInvadeSafe: boolean;
}

export interface NemesisState {
  records: NemesisRecord[];
  gameTurn: number;
  totalNemesesCreated: number;
}
```

Add `isNemesis` optional field to the existing `TankEntity` interface, after the `aiTarget` field:

```typescript
  aiTarget?: string;
  isNemesis?: boolean;         // NEW: marks this tank as a nemesis ace
```

**File: `src/utils/constants.ts`**

Add nemesis constants after the existing `SCORE_PER_KILL` constant block:

```typescript
// --- Nemesis System ---
export const NEMESIS_COLOR = '#FF2222';
export const NEMESIS_MAX_ACTIVE = 4;
export const NEMESIS_RANKS: readonly string[] = ['RAIDER', 'MARAUDER', 'WARLORD', 'OVERLORD', 'TYRANT'];
export const NEMESIS_XP_THRESHOLDS = [0, 100, 250, 500, 900] as const;
export const NEMESIS_ENTOURAGE_SIZE = [0, 1, 2, 3, 4] as const;
export const NEMESIS_STAT_BONUS = [0, 0.15, 0.30, 0.50, 0.75] as const;
export const NEMESIS_XP_PER_KILL = 100;
export const NEMESIS_XP_PER_TURN = 25;
export const NEMESIS_DETECTION_MULT = 1.5;
export const NEMESIS_SPEED_SCALE = 0.5;  // speed bonus = stat_bonus * this
export const NEMESIS_ROAM_BASE_RADIUS = 2;
export const NEMESIS_TAUNT_DELAY_MS = 1500;
export const NEMESIS_DEFEAT_SCORE_MULT = 2;
```

### Step 2: Create NemesisSystem Module

**File: `src/game/NemesisSystem.ts`** (NEW FILE)

This is the core nemesis logic module. Pure functions + localStorage persistence. No React dependencies.

```typescript
import type { NemesisRecord, NemesisState, NemesisRank, TankEntity, Vec3 } from '../utils/types';
import {
  NEMESIS_COLOR,
  NEMESIS_MAX_ACTIVE,
  NEMESIS_RANKS,
  NEMESIS_XP_THRESHOLDS,
  NEMESIS_ENTOURAGE_SIZE,
  NEMESIS_STAT_BONUS,
  NEMESIS_XP_PER_KILL,
  NEMESIS_XP_PER_TURN,
  NEMESIS_ROAM_BASE_RADIUS,
  SCORE_PER_KILL,
  NEMESIS_DEFEAT_SCORE_MULT,
} from '../utils/constants';
import { coordsToSeed, mulberry32, deriveValues } from '../world/WorldSeed';

// ═══════════════════════════════════════════════════════════════
// PERSISTENCE
// ═══════════════════════════════════════════════════════════════

const STORAGE_KEY = 'rt-nemesis';

const DEFAULT_STATE: NemesisState = {
  records: [],
  gameTurn: 0,
  totalNemesesCreated: 0,
};

export function loadNemesisState(): NemesisState {
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    return raw ? JSON.parse(raw) : { ...DEFAULT_STATE };
  } catch {
    return { ...DEFAULT_STATE };
  }
}

export function saveNemesisState(state: NemesisState): void {
  try {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(state));
  } catch { /* quota exceeded */ }
}

// ═══════════════════════════════════════════════════════════════
// NAME GENERATION
// ═══════════════════════════════════════════════════════════════

const COMMANDER_TITLES = [
  'COMMANDER', 'CAPTAIN', 'WARDEN', 'COLONEL', 'MARSHAL',
  'GENERAL', 'BARON', 'OVERSEER', 'ENFORCER', 'EXECUTIONER',
];

const COMMANDER_NAMES = [
  'VEXAR', 'DRAKOS', 'KAINE', 'SOLEN', 'RHEX',
  'TYGOR', 'NEXUS', 'CRYON', 'VOLTIS', 'ASHKOR',
  'BRYNN', 'ZERITH', 'MORDAK', 'PYLAX', 'STRIX',
];

export function generateNemesisName(seed: number): string {
  const rng = mulberry32(seed);
  const title = COMMANDER_TITLES[Math.floor(rng() * COMMANDER_TITLES.length)];
  const name = COMMANDER_NAMES[Math.floor(rng() * COMMANDER_NAMES.length)];
  return `${title} ${name}`;
}

// ═══════════════════════════════════════════════════════════════
// LEVEL / RANK CALCULATION
// ═══════════════════════════════════════════════════════════════

export function computeLevel(xp: number): number {
  let level = 1;
  for (let i = NEMESIS_XP_THRESHOLDS.length - 1; i >= 0; i--) {
    if (xp >= NEMESIS_XP_THRESHOLDS[i]) {
      level = i + 1;
      break;
    }
  }
  return level;
}

export function computeRank(level: number): NemesisRank {
  const idx = Math.min(level - 1, NEMESIS_RANKS.length - 1);
  return NEMESIS_RANKS[idx] as NemesisRank;
}

// ═══════════════════════════════════════════════════════════════
// STAT COMPUTATION (for spawning as TankEntity)
// ═══════════════════════════════════════════════════════════════

export function computeNemesisEntityStats(record: NemesisRecord): Partial<TankEntity> {
  const bonus = NEMESIS_STAT_BONUS[record.level - 1] ?? 0;
  return {
    health: Math.round(record.baseHealth * (1 + bonus)),
    maxHealth: Math.round(record.baseHealth * (1 + bonus)),
    shield: Math.round(50 * (1 + bonus)),
    maxShield: Math.round(50 * (1 + bonus)),
    shieldRechargeRate: Math.round(3 * (1 + bonus * 0.5)),
    armor: Math.min(10, 5 + record.level),
    damageBuff: record.baseDamage * (1 + bonus),
    damageBuffTimeRemaining: Infinity,
    speedBuff: record.baseSpeed * (1 + bonus * 0.5),
    speedBuffTimeRemaining: Infinity,
    isNemesis: true,
    color: NEMESIS_COLOR,
  };
}

/**
 * Create a full TankEntity for a nemesis at the given spawn position.
 */
export function createNemesisTankEntity(record: NemesisRecord, spawn: Vec3): TankEntity {
  const stats = computeNemesisEntityStats(record);
  return {
    id: record.id,
    position: { ...spawn },
    rotation: Math.random() * Math.PI * 2,
    turretRotation: 0,
    velocity: { x: 0, y: 0, z: 0 },
    health: stats.health!,
    maxHealth: stats.maxHealth!,
    shield: stats.shield!,
    maxShield: stats.maxShield!,
    shieldRechargeRate: stats.shieldRechargeRate!,
    armor: stats.armor!,
    speed: 0,
    isPlayer: false,
    color: NEMESIS_COLOR,
    currentWeapon: 'precision',
    boostCooldownRemaining: 0,
    isBoosting: false,
    boostTimeRemaining: 0,
    isDrifting: false,
    driftAngle: 0,
    driftSpeedFloor: 0,
    weaponCooldowns: { precision: 0, devastator: 0, suppressor: 0, shredder: 0, seeker: 0 },
    overheatLevel: 0,
    isOverheated: false,
    aiState: 'aggressive_chase',
    isNemesis: true,
    damageBuff: stats.damageBuff!,
    damageBuffTimeRemaining: Infinity,
    speedBuff: stats.speedBuff!,
    speedBuffTimeRemaining: Infinity,
    slowDebuff: 1,
    slowDebuffTimeRemaining: 0,
    loadoutSpeedMult: 1,
    loadoutDamageMult: 1,
    currentTargetId: null,
    invulnerableUntil: 0,
  };
}

/**
 * Create entourage (escort) tanks for a nemesis.
 * Returns an array of regular enemy TankEntities with slightly boosted stats.
 */
export function createNemesisEntourage(
  record: NemesisRecord,
  baseSpawn: Vec3,
  valleyId: string,
  startIndex: number,
): TankEntity[] {
  const count = NEMESIS_ENTOURAGE_SIZE[record.level - 1] ?? 0;
  if (count === 0) return [];

  const escorts: TankEntity[] = [];
  const bonus = NEMESIS_STAT_BONUS[record.level - 1] ?? 0;
  const escortBonus = bonus * 0.5; // escorts are weaker than the nemesis itself

  for (let i = 0; i < count; i++) {
    const angle = (i / count) * Math.PI * 2;
    const radius = 15 + Math.random() * 10;
    const spawnPos: Vec3 = {
      x: baseSpawn.x + Math.cos(angle) * radius,
      y: baseSpawn.y,
      z: baseSpawn.z + Math.sin(angle) * radius,
    };

    escorts.push({
      id: `escort-${record.id}-${startIndex + i}`,
      position: spawnPos,
      rotation: Math.random() * Math.PI * 2,
      turretRotation: 0,
      velocity: { x: 0, y: 0, z: 0 },
      health: Math.round(100 * (1 + escortBonus)),
      maxHealth: Math.round(100 * (1 + escortBonus)),
      shield: Math.round(30 * (1 + escortBonus)),
      maxShield: Math.round(30 * (1 + escortBonus)),
      shieldRechargeRate: 3,
      armor: Math.min(8, 3 + record.level),
      speed: 0,
      isPlayer: false,
      color: '#CC4444', // darker red for escorts
      currentWeapon: 'precision',
      boostCooldownRemaining: 0,
      isBoosting: false,
      boostTimeRemaining: 0,
      isDrifting: false,
      driftAngle: 0,
      driftSpeedFloor: 0,
      weaponCooldowns: { precision: 0, devastator: 0, suppressor: 0, shredder: 0, seeker: 0 },
      overheatLevel: 0,
      isOverheated: false,
      aiState: 'aggressive_chase',
      damageBuff: 1 + escortBonus,
      damageBuffTimeRemaining: 0,
      speedBuff: 1,
      speedBuffTimeRemaining: 0,
      slowDebuff: 1,
      slowDebuffTimeRemaining: 0,
      loadoutSpeedMult: 1,
      loadoutDamageMult: 1,
      currentTargetId: null,
      invulnerableUntil: 0,
    });
  }

  return escorts;
}

// ═══════════════════════════════════════════════════════════════
// NEMESIS CREATION (on player death)
// ═══════════════════════════════════════════════════════════════

/**
 * Hash a string to a positive integer (for seeding).
 */
function hashStringToInt(str: string): number {
  let hash = 0;
  for (let i = 0; i < str.length; i++) {
    const chr = str.charCodeAt(i);
    hash = ((hash << 5) - hash) + chr;
    hash |= 0;
  }
  return Math.abs(hash);
}

/**
 * Called when the player is killed.
 * killerId: the enemy ID that dealt the lethal blow.
 * killerEntity: the TankEntity of the killer (for base stats).
 * Returns the updated NemesisState and the promoted/created NemesisRecord (or null).
 */
export function onPlayerKilled(
  state: NemesisState,
  killerId: string,
  killerEntity: TankEntity,
  playerValleyX: number,
  playerValleyZ: number,
): { newState: NemesisState; promotedNemesis: NemesisRecord | null } {
  const newState = { ...state, records: state.records.map(r => ({ ...r })) };

  // Check if killer is already a nemesis
  const existingIdx = newState.records.findIndex(r => r.id === killerId && !r.defeated);
  if (existingIdx >= 0) {
    // Promote existing nemesis
    const rec = newState.records[existingIdx];
    rec.wins += 1;
    rec.xp += NEMESIS_XP_PER_KILL;
    rec.level = computeLevel(rec.xp);
    rec.rank = computeRank(rec.level);
    rec.entourageSize = NEMESIS_ENTOURAGE_SIZE[rec.level - 1] ?? 0;
    rec.canInvadeSafe = rec.level >= 5;
    rec.lastEncounterTurn = newState.gameTurn;
    rec.dialogueIndex = Math.min(rec.dialogueIndex + 1, 24);
    saveNemesisState(newState);
    return { newState, promotedNemesis: rec };
  }

  // Check active nemesis count
  const activeCount = newState.records.filter(r => !r.defeated).length;
  if (activeCount >= NEMESIS_MAX_ACTIVE) {
    // No room — give XP to highest-level active nemesis
    const sorted = newState.records
      .filter(r => !r.defeated)
      .sort((a, b) => b.xp - a.xp);
    if (sorted.length > 0) {
      const top = newState.records.find(r => r.id === sorted[0].id)!;
      top.xp += 50;
      top.level = computeLevel(top.xp);
      top.rank = computeRank(top.level);
      top.entourageSize = NEMESIS_ENTOURAGE_SIZE[top.level - 1] ?? 0;
      top.canInvadeSafe = top.level >= 5;
    }
    saveNemesisState(newState);
    return { newState, promotedNemesis: null };
  }

  // Create new nemesis from the killer
  newState.totalNemesesCreated += 1;
  const nameSeed = hashStringToInt(killerId) + newState.gameTurn * 7919 + newState.totalNemesesCreated;

  const newRecord: NemesisRecord = {
    id: `nemesis-${newState.totalNemesesCreated}`,
    name: generateNemesisName(nameSeed),
    rank: 'RAIDER',
    wins: 1,
    xp: NEMESIS_XP_PER_KILL,
    level: 1,
    baseHealth: killerEntity.maxHealth,
    baseDamage: killerEntity.damageBuff,
    baseSpeed: 1,
    currentValleyX: playerValleyX,
    currentValleyZ: playerValleyZ,
    lastEncounterTurn: newState.gameTurn,
    dialogueIndex: 8, // starts with a gloat line
    defeated: false,
    defeatedTurn: null,
    createdTurn: newState.gameTurn,
    color: NEMESIS_COLOR,
    entourageSize: 0,
    canInvadeSafe: false,
  };

  newState.records.push(newRecord);
  saveNemesisState(newState);
  return { newState, promotedNemesis: newRecord };
}

// ═══════════════════════════════════════════════════════════════
// NEMESIS DEFEAT (player kills a nemesis)
// ═══════════════════════════════════════════════════════════════

/**
 * Called when a nemesis tank is destroyed.
 * Returns updated state, the defeated record, and bonus score to award.
 */
export function onNemesisDefeated(
  state: NemesisState,
  nemesisId: string,
): { newState: NemesisState; defeatedRecord: NemesisRecord | null; bonusScore: number } {
  const newState = { ...state, records: state.records.map(r => ({ ...r })) };
  const idx = newState.records.findIndex(r => r.id === nemesisId);

  if (idx < 0) return { newState, defeatedRecord: null, bonusScore: 0 };

  const rec = newState.records[idx];
  rec.defeated = true;
  rec.defeatedTurn = newState.gameTurn;

  const bonusScore = SCORE_PER_KILL * rec.level * NEMESIS_DEFEAT_SCORE_MULT;

  saveNemesisState(newState);
  return { newState, defeatedRecord: rec, bonusScore };
}

// ═══════════════════════════════════════════════════════════════
// ROAMING — advance all nemesis positions on valley transition
// ═══════════════════════════════════════════════════════════════

/**
 * Advance the game turn and update all active nemesis valley positions.
 */
export function advanceNemesisRoaming(
  state: NemesisState,
  worldSeed: number,
): NemesisState {
  const newState = {
    ...state,
    gameTurn: state.gameTurn + 1,
    records: state.records.map(r => {
      if (r.defeated) return { ...r };

      // Award per-turn XP
      const newXp = r.xp + NEMESIS_XP_PER_TURN;
      const newLevel = computeLevel(newXp);
      const newRank = computeRank(newLevel);

      // Compute new roaming position
      const roamRadius = NEMESIS_ROAM_BASE_RADIUS + newLevel;
      const seed = coordsToSeed(
        hashStringToInt(r.id),
        state.gameTurn + 1,
        0,
        worldSeed + 800000,
      );
      const rng = mulberry32(seed);
      const vx = Math.floor((rng() - 0.5) * 2 * roamRadius);
      const vz = Math.floor((rng() - 0.5) * 2 * roamRadius);

      // TYRANT can invade (0,0); lower ranks skip it
      const canInvade = newLevel >= 5;
      let finalVx = vx;
      let finalVz = vz;
      if (finalVx === 0 && finalVz === 0 && !canInvade) {
        finalVx = 1; // push out of home base
      }

      return {
        ...r,
        xp: newXp,
        level: newLevel,
        rank: newRank,
        entourageSize: NEMESIS_ENTOURAGE_SIZE[newLevel - 1] ?? 0,
        canInvadeSafe: canInvade,
        currentValleyX: finalVx,
        currentValleyZ: finalVz,
      };
    }),
  };

  saveNemesisState(newState);
  return newState;
}

// ═══════════════════════════════════════════════════════════════
// QUERY HELPERS
// ═══════════════════════════════════════════════════════════════

/**
 * Get all active (non-defeated) nemesis records for a given valley.
 */
export function getNemesesInValley(
  state: NemesisState,
  valleyX: number,
  valleyZ: number,
): NemesisRecord[] {
  return state.records.filter(
    r => !r.defeated && r.currentValleyX === valleyX && r.currentValleyZ === valleyZ
  );
}

/**
 * Get the single highest-level active nemesis (for guaranteed encounters).
 */
export function getTopNemesis(state: NemesisState): NemesisRecord | null {
  const active = state.records.filter(r => !r.defeated);
  if (active.length === 0) return null;
  return active.reduce((a, b) => (b.xp > a.xp ? b : a));
}

/**
 * Get all active nemesis records.
 */
export function getActiveNemeses(state: NemesisState): NemesisRecord[] {
  return state.records.filter(r => !r.defeated);
}
```

### Step 3: Create Nemesis Monologues Data

**File: `src/data/nemesisMonologues.ts`** (NEW FILE)

```typescript
// ============================================================
// NEMESIS MONOLOGUES -- 25 evil tank commander voice lines
// ============================================================
// Three categories: taunts (encounter), gloats (player killed),
// defeat monologues (nemesis killed). Selected by dialogueIndex.

export const NEMESIS_MONOLOGUES: string[] = [
  // --- TAUNTS (on spawn/encounter, indices 0-7) ---
  'Your signal is weak, commander. Just like your armor.',
  "I've been tracking you across three sectors. Nowhere left to run.",
  "Every valley you enter, I'm already there. Waiting.",
  'My targeting systems have your signature memorized.',
  'You think these mountains will save you? I own this grid.',
  'Last time was just a warm-up. This time, I finish it.',
  "Your wingman can't save you from what's coming.",
  "I've upgraded since our last dance. Have you?",

  // --- GLOATS (on killing the player, indices 8-15) ---
  'Another notch on my barrel. You never learn.',
  "Predictable. Your tactics haven't changed since our first meeting.",
  "That's {wins} times now. Are you even trying?",
  'Your wreckage makes excellent cover. Thanks for that.',
  'Uploading your destruction to the grid. Everyone will see.',
  "They'll need to send someone better. You're not enough.",
  'I almost felt that last shot. Almost.',
  'Stay down this time. The grid has no place for the weak.',

  // --- DEFEAT MONOLOGUES (on nemesis death, indices 16-24) ---
  'Impossible... my systems were... superior...',
  'This changes nothing. The grid will send another.',
  "You got lucky. Next time there won't be luck.",
  'My data... will be... uploaded... someone will finish... this...',
  'So this is how a machine dies. Cold. In the dark.',
  "Fine. You've earned this sector. For now.",
  "I underestimated you, commander. I won't make that mistake in my next cycle.",
  'The grid... remembers... everything...',
  'Tell them... tell them TYRANT fell fighting. Not running. Never running.',
];

/**
 * Get a monologue line, replacing template variables.
 */
export function getMonologue(index: number, vars?: { wins?: number }): string {
  const clamped = Math.max(0, Math.min(index, NEMESIS_MONOLOGUES.length - 1));
  let line = NEMESIS_MONOLOGUES[clamped];
  if (vars?.wins !== undefined) {
    line = line.replace('{wins}', String(vars.wins));
  }
  return line;
}

/**
 * Get a random taunt index (0-7).
 */
export function getRandomTauntIndex(seed: number): number {
  return Math.abs(seed) % 8;
}

/**
 * Get a random gloat index (8-15).
 */
export function getRandomGloatIndex(seed: number): number {
  return 8 + (Math.abs(seed) % 8);
}

/**
 * Get a defeat monologue index (16-24).
 */
export function getDefeatMonologueIndex(seed: number): number {
  return 16 + (Math.abs(seed) % 9);
}
```

### Step 4: Integrate Nemesis into NPC Dialog System

**File: `src/data/npcDialogs.ts`**

Add `'nemesis'` to the NPCType union and color map:

```typescript
// CHANGE LINE 13:
export type NPCType = 'merchant' | 'mechanic' | 'scout' | 'quest_giver' | 'nemesis';

// CHANGE LINE 15-20 (add nemesis entry):
export const NPC_TYPE_COLORS: Record<NPCType, string> = {
  merchant: '#FFD700',
  mechanic: '#FF8C00',
  scout: '#00FFFF',
  quest_giver: '#FF00FF',
  nemesis: '#FF2222',       // NEW: crimson for hacked comms
};
```

Add a nemesis dialog factory function at the bottom of the file:

```typescript
import type { DialogTree } from '../utils/types';
import { getMonologue } from './nemesisMonologues';

/**
 * Create a one-shot dialog tree for a nemesis comms message.
 * Uses the "hacked comms" aesthetic: speaker is UNKNOWN (or name if
 * encountered 2+ times), single text node, no choices.
 */
export function createNemesisDialog(
  nemesisName: string,
  wins: number,
  dialogueIndex: number,
  revealName: boolean,
): DialogTree {
  const line = getMonologue(dialogueIndex, { wins });
  return {
    id: `nemesis-comms-${Date.now()}`,
    startNodeId: 'msg',
    nodes: {
      msg: {
        speaker: revealName ? nemesisName : 'UNKNOWN',
        text: line,
        nextNodeId: null,
      },
    },
  };
}
```

### Step 5: Add Nemesis Asset Entries

**File: `src/data/npcAssets.ts`**

Add nemesis entry to `NPC_ASSET_URLS` (after `quest_giver`). This will use a shared "hacked comms" static image. If the image file doesn't exist yet, the `DialogOverlay` will fall back to the `ProceduralPortrait`:

```typescript
const NPC_ASSET_URLS: Record<NPCType, Record<NPCImageMode, string>> = {
  merchant: { /* existing */ },
  mechanic: { /* existing */ },
  scout: { /* existing */ },
  quest_giver: { /* existing */ },
  nemesis: {
    portrait: `${BASE}npc/portraits/nemesis_portrait.png`,
    chat: `${BASE}npc/chat/nemesis_chat.png`,
  },
};
```

### Step 6: Add Nemesis State to GameStore

**File: `src/game/GameState.ts`**

Add nemesis state and actions to the store interface and implementation:

```typescript
// In the GameStore interface, add after the vehicleMode section:

  // --- Nemesis System ---
  nemesisState: NemesisState;
  setNemesisState: (state: NemesisState) => void;
  triggerNemesisDialog: (name: string, wins: number, dialogueIndex: number, revealName: boolean) => void;
```

Add imports at top:

```typescript
import type { NemesisState } from '../utils/types';
import { loadNemesisState, saveNemesisState } from './NemesisSystem';
```

In the store creation, add:

```typescript
  // --- Nemesis System ---
  nemesisState: loadNemesisState(),
  setNemesisState: (state) => {
    saveNemesisState(state);
    set({ nemesisState: state });
  },
  triggerNemesisDialog: (name, wins, dialogueIndex, revealName) => {
    // Dynamically import to avoid circular deps
    const { createNemesisDialog } = require('../data/npcDialogs');
    const tree = createNemesisDialog(name, wins, dialogueIndex, revealName);
    set({
      activeDialog: tree,
      activeDialogNode: tree.startNodeId,
      activeNPCType: 'nemesis',
      activeNPCSeed: Math.floor(Math.random() * 99999),
    });
  },
```

**Note on `triggerNemesisDialog`:** This intentionally does NOT set `isPaused: true` — nemesis taunts play during live gameplay as an overlay while combat continues. The player can click to dismiss or wait for auto-advance.

### Step 7: Integrate Nemesis into Enemy Spawning (OpenWorldScreen)

**File: `src/screens/OpenWorldScreen.tsx`**

This is the largest integration point. Changes occur in the `useFrame` valley transition block.

**7a. Add imports:**

```typescript
import {
  loadNemesisState,
  getNemesesInValley,
  advanceNemesisRoaming,
  createNemesisTankEntity,
  createNemesisEntourage,
  onPlayerKilled,
  onNemesisDefeated,
} from '../game/NemesisSystem';
import { getMonologue, getRandomTauntIndex } from '../data/nemesisMonologues';
import {
  NEMESIS_COLOR,
  NEMESIS_TAUNT_DELAY_MS,
  NEMESIS_DEFEAT_SCORE_MULT,
  SCORE_PER_KILL,
} from '../utils/constants';
```

**7b. In the valley transition block** (around line 393, after `store.enterValley(valley)`):

After the existing enemy spawning code (`store.setEnemies(valleyEnemies)`), add nemesis spawning:

```typescript
      // --- Nemesis roaming & spawning ---
      let nemState = store.nemesisState;
      nemState = advanceNemesisRoaming(nemState, store.worldSeed);
      store.setNemesisState(nemState);

      const nemesesHere = getNemesesInValley(nemState, valley.gridX, valley.gridZ);
      if (nemesesHere.length > 0) {
        const currentEnemies = useGameStore.getState().enemies;
        const nemesisTanks: TankEntity[] = [];

        for (const nemRec of nemesesHere) {
          // Spawn nemesis at a random position in the valley
          const rng = deriveValues(valley.gridX, valley.gridZ, nemRec.createdTurn, store.worldSeed + 900000, 3);
          const originX = valley.gridX * VALLEY_SIZE;
          const originZ = valley.gridZ * VALLEY_SIZE;
          const nemSpawn: Vec3 = {
            x: originX + (rng[0] - 0.5) * VALLEY_SIZE * 0.5,
            y: 0,
            z: originZ + (rng[1] - 0.5) * VALLEY_SIZE * 0.5,
          };
          nemSpawn.y = getTerrainHeight(nemSpawn.x, nemSpawn.z, store.worldSeed);

          const nemTank = createNemesisTankEntity(nemRec, nemSpawn);
          nemesisTanks.push(nemTank);

          // Spawn entourage
          const escorts = createNemesisEntourage(nemRec, nemSpawn, valley.id, currentEnemies.length + nemesisTanks.length);
          for (const escort of escorts) {
            escort.position.y = getTerrainHeight(escort.position.x, escort.position.z, store.worldSeed);
          }
          nemesisTanks.push(...escorts);

          // Trigger taunt dialog after a short delay
          const tauntIdx = getRandomTauntIndex(nemRec.createdTurn + nemState.gameTurn);
          const revealName = nemRec.wins >= 2;
          setTimeout(() => {
            store.triggerNemesisDialog(nemRec.name, nemRec.wins, tauntIdx, revealName);
          }, NEMESIS_TAUNT_DELAY_MS);
        }

        // Append nemesis tanks to existing valley enemies
        store.setEnemies([...currentEnemies, ...nemesisTanks]);
      }
```

**7c. In the `processDamageEvents` function** — handle nemesis death:

After the existing enemy death block (`if (damaged.health <= 0)` around line 258), add:

```typescript
        if (damaged.health <= 0) {
          // ... existing death code (updateEnemy, incrementEnemiesDestroyed, explosion, score) ...

          // --- Nemesis defeat check ---
          if (damaged.isNemesis) {
            const nemState = useGameStore.getState().nemesisState;
            const { newState, defeatedRecord, bonusScore } = onNemesisDefeated(nemState, damaged.id);
            store.setNemesisState(newState);
            if (bonusScore > 0) {
              store.addScore(bonusScore);
            }
            // Trigger defeat monologue
            if (defeatedRecord) {
              getExplosionManager().trigger(damaged.position, 'LARGE'); // big explosion
              setTimeout(() => {
                store.triggerNemesisDialog(
                  defeatedRecord.name,
                  defeatedRecord.wins,
                  defeatedRecord.dialogueIndex,
                  true, // always reveal name on death
                );
              }, 800);
            }
          }
        }
```

**7d. In the player death block** — handle nemesis creation:

After the existing player death code (`if (damaged.health <= 0)` around line 232), BEFORE the `return { player: null, drops }`, add:

```typescript
      if (damaged.health <= 0) {
        audioManager.playSFX('death');
        getEngineAudio().stop();
        getExplosionManager().trigger(damaged.position, 'LARGE');
        triggerCameraShake(EXPLOSION_CONFIG.LARGE.cameraShake);

        // --- Nemesis creation on player death ---
        const killerEnemy = currentEnemies.find(e => e.id === event.source && e.health > 0);
        if (killerEnemy) {
          const valley = useGameStore.getState().currentValley;
          const nemState = useGameStore.getState().nemesisState;
          const { newState, promotedNemesis } = onPlayerKilled(
            nemState,
            killerEnemy.id,
            killerEnemy,
            valley?.gridX ?? 0,
            valley?.gridZ ?? 0,
          );
          store.setNemesisState(newState);

          // Trigger gloat dialog after death explosion
          if (promotedNemesis) {
            const gloatIdx = promotedNemesis.wins <= 1
              ? promotedNemesis.dialogueIndex  // initial gloat (index 8)
              : Math.min(promotedNemesis.dialogueIndex, 15); // escalating gloats
            setTimeout(() => {
              store.triggerNemesisDialog(
                promotedNemesis.name,
                promotedNemesis.wins,
                gloatIdx,
                promotedNemesis.wins >= 2,
              );
            }, NEMESIS_TAUNT_DELAY_MS);
          }
        }

        store.endGame(false);
        return { player: null, drops };
      }
```

Note: `currentEnemies` is the local enemies array already available in `processDamageEvents`.

### Step 8: Integrate Nemesis into AI System

**File: `src/game/AISystem.ts`**

Add a nemesis-aware override block at the start of `updateEnemyAI`, before the existing state transition logic:

```typescript
import { NEMESIS_DETECTION_MULT, NEMESIS_SPEED_SCALE, NEMESIS_STAT_BONUS } from '../utils/constants';
```

Then inside `updateEnemyAI`, after `let state = enemy.aiState ?? 'patrol';`:

```typescript
  // --- Nemesis overrides ---
  let effectiveDetectionRange = AI_DEFAULTS.detectionRange;
  let effectiveRetreatThreshold = AI_DEFAULTS.retreatThreshold;
  let effectiveMoveSpeed = AI_DEFAULTS.moveSpeed;

  if (enemy.isNemesis) {
    effectiveDetectionRange = AI_DEFAULTS.detectionRange * NEMESIS_DETECTION_MULT;
    effectiveRetreatThreshold = 0; // nemeses never flee
    // Nemesis speed boost is already baked into speedBuff, but also boost base AI move speed
    effectiveMoveSpeed = AI_DEFAULTS.moveSpeed * 1.2;
  }
```

Then replace all references to `AI_DEFAULTS.detectionRange` with `effectiveDetectionRange`, `AI_DEFAULTS.retreatThreshold` with `effectiveRetreatThreshold`, and update the speed calculation at line 165:

```typescript
  // CHANGE line 61:
  if (healthPct <= effectiveRetreatThreshold && state !== 'flee' && state !== 'seek_cover') {

  // CHANGE line 65:
  } else if (dist < effectiveDetectionRange && state === 'patrol') {

  // CHANGE line 67:
  } else if (dist > effectiveDetectionRange * 1.2 && state !== 'patrol' && state !== 'flee') {

  // CHANGE line 76:
  if (dist < effectiveDetectionRange) {

  // CHANGE line 165:
  const speed = state === 'strafe_attack' ? AI_DEFAULTS.strafeSpeed : effectiveMoveSpeed;
```

Also enhance weapon selection for high-rank nemeses. In the `strafe_attack` case (around line 117), add:

```typescript
      // Nemesis weapon selection: use seeker at long range, devastator at medium
      let weapon: WeaponType;
      if (enemy.isNemesis && dist > 60) {
        weapon = 'seeker';
      } else if (enemy.isNemesis && dist > 25 && dist <= 40) {
        weapon = Math.random() > 0.4 ? 'devastator' : chooseWeapon(dist);
      } else {
        weapon = chooseWeapon(dist);
      }
      fireRequest = { weapon };
```

### Step 9: HUD Nemesis Indicator

**File: `src/components/ui/HUD.tsx`**

Add a nemesis detection indicator. Add a selector for enemies:

```typescript
const enemies = useGameStore((s) => s.enemies);
```

Then add the indicator element inside the HUD JSX, after the existing target indicator block:

```typescript
      {/* Nemesis Alert */}
      {(() => {
        const nemTank = enemies.find(e => e.isNemesis && e.health > 0);
        if (!nemTank || isOnFoot) return null;
        const nemState = useGameStore.getState().nemesisState;
        const nemRec = nemState.records.find(r => r.id === nemTank.id && !r.defeated);
        if (!nemRec) return null;
        const hpPct = Math.round((nemTank.health / nemTank.maxHealth) * 100);
        return (
          <div style={{
            position: 'absolute',
            top: 140,
            left: '50%',
            transform: 'translateX(-50%)',
            textAlign: 'center',
            fontFamily: "'Courier New', monospace",
            zIndex: 20,
            pointerEvents: 'none',
          }}>
            <div style={{
              color: '#FF2222',
              fontSize: '0.75rem',
              fontWeight: 'bold',
              textShadow: '0 0 15px #FF222280',
              letterSpacing: '0.15em',
              animation: 'pulse 1.5s ease-in-out infinite',
            }}>
              ACE DETECTED: {nemRec.rank} {nemRec.name}
            </div>
            <div style={{
              marginTop: 4,
              width: 120,
              height: 4,
              background: '#333',
              borderRadius: 2,
              margin: '4px auto 0',
              overflow: 'hidden',
            }}>
              <div style={{
                width: `${hpPct}%`,
                height: '100%',
                background: hpPct > 50 ? '#FF2222' : hpPct > 25 ? '#FF8800' : '#FF0000',
                transition: 'width 0.3s',
              }} />
            </div>
            <div style={{
              color: '#FF222280',
              fontSize: '0.55rem',
              marginTop: 2,
            }}>
              HULL {hpPct}%
            </div>
          </div>
        );
      })()}
```

Add the CSS `pulse` keyframe. Check if it already exists in the codebase — if not, add to the HUD component's style injection or inline:

```css
@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.5; }
}
```

### Step 10: Add isNemesis to All TankEntity Literals

Every location that creates a `TankEntity` literal needs the optional `isNemesis` field. Since it's optional (`isNemesis?: boolean`), existing code that doesn't specify it defaults to `undefined` (falsy), which is correct. **No changes needed** to existing TankEntity literals — they will naturally evaluate `enemy.isNemesis` as `undefined` (falsy).

However, if TypeScript strict mode complains, explicitly add `isNemesis: false` to:
- `createPlayer()` in GameState.ts
- `createEnemy()` in GameState.ts
- `createValleyEnemy()` in OpenWorldScreen.tsx
- `PreviewTank` entity in TheBunkerScreen.tsx
- `MENU_TANK` in MainMenuScreen.tsx

### Step 11: Wire `triggerNemesisDialog` Without Circular Dependencies

The `triggerNemesisDialog` store action calls `createNemesisDialog` from `npcDialogs.ts`. To avoid circular dependency issues (GameState imports from npcDialogs which imports types):

**Option A (preferred):** Use dynamic import:
```typescript
triggerNemesisDialog: async (name, wins, dialogueIndex, revealName) => {
  const { createNemesisDialog } = await import('../data/npcDialogs');
  const tree = createNemesisDialog(name, wins, dialogueIndex, revealName);
  set({
    activeDialog: tree,
    activeDialogNode: tree.startNodeId,
    activeNPCType: 'nemesis',
    activeNPCSeed: Math.floor(Math.random() * 99999),
  });
},
```

**Option B:** Move `createNemesisDialog` into `nemesisMonologues.ts` (which has no circular dep risk) and import from there.

### Step 12: Build Verification

Run the following to verify clean compilation:

```bash
npx tsc --noEmit
npm run build
```

Expected: 0 errors, ~692 modules (2 new files), ~1,215 KB bundle.

---

## Testing Scenarios

### T1. Nemesis Creation
1. Enter a combat valley, engage enemies
2. Allow an enemy to kill the player
3. Verify `rt-nemesis` localStorage key now contains a record with `wins: 1`, `rank: 'RAIDER'`
4. Verify the gloat dialog appears after death with "UNKNOWN" speaker

### T2. Nemesis Encounter
1. After being killed, restart and travel to valleys
2. When entering the valley where the nemesis roams, verify:
   - A crimson tank spawns with `isNemesis: true`
   - The taunt dialog fires after 1.5s delay
   - The HUD shows "ACE DETECTED: RAIDER [NAME]"
3. Verify the nemesis is more aggressive (immediately chases)

### T3. Nemesis Promotion
1. Get killed by the same nemesis again
2. Verify: `wins: 2`, `xp: 200`, `rank: 'MARAUDER'`, `entourageSize: 1`
3. On next encounter, verify 1 escort spawns with the nemesis
4. Verify the gloat dialog now reveals the nemesis name (wins >= 2)

### T4. Nemesis Defeat
1. Destroy a nemesis tank in combat
2. Verify: `defeated: true` in the record
3. Verify bonus score awarded: `SCORE_PER_KILL * level * 2`
4. Verify defeat monologue plays via dialog overlay
5. Verify the nemesis no longer spawns in subsequent valleys

### T5. Max Active Cap
1. Get killed by 4 different enemies in 4 different valleys
2. Verify exactly 4 nemesis records exist, all active
3. Get killed by a 5th different enemy
4. Verify no 5th nemesis created; instead the top nemesis got +50 XP

### T6. TYRANT Invasion
1. Use dev tools to set a nemesis to `xp: 900`, `level: 5`, `rank: 'TYRANT'`
2. Allow roaming to tick; eventually the tyrant should be placed at `(0,0)`
3. Enter home base valley (0,0) and verify the TYRANT spawns with 4 escorts
4. Verify warning dialog fires about hostile ace in safe zone

### T7. Persistence
1. Create nemeses across multiple play sessions
2. Close browser, reopen
3. Verify all nemesis records restored from `rt-nemesis` localStorage
4. Verify roaming continues from the persisted `gameTurn`

### T8. Buff Timer Integrity
1. Spawn a nemesis (has `damageBuffTimeRemaining: Infinity`)
2. Wait 60+ seconds of gameplay
3. Verify the nemesis's `damageBuff` has NOT decayed to 1 (the `Infinity` time remaining prevents the timer logic from resetting)

---

## Style Guide

### Nemesis Visual Identity
- **Tank color:** `#FF2222` (crimson red) — distinct from all existing enemy colors
- **Escort color:** `#CC4444` (darker red) — identifies entourage members
- **HUD text:** Crimson with `textShadow: '0 0 15px #FF222280'` glow
- **Dialog border:** `#FF2222` border on the toast bubble (via `NPC_TYPE_COLORS.nemesis`)
- **Dialog speaker tag:** Crimson background with dark text

### Naming Convention
- All nemesis constants prefixed with `NEMESIS_`
- All nemesis functions prefixed with `nemesis` or contain `Nemesis`
- Storage key: `rt-nemesis` (follows existing `rt-` prefix pattern)
- Entity IDs: `nemesis-{n}` for nemeses, `escort-{nemesisId}-{n}` for entourage

---

## Performance Goals

- **Zero per-frame overhead** when no nemesis is in the current valley
- Nemesis spawning adds at most 5 additional entities (1 nemesis + 4 escorts) to the enemy pool, well within the existing `MAX_ENEMIES = 20` cap
- `localStorage` read/write is O(n) where n = number of nemesis records (max ~10-20 over a long campaign)
- No new render passes, textures, or 3D geometry — nemeses use existing `<Tank>` component
- Dialog overlay is an existing lightweight toast — no modal or fullscreen rendering

---

## File Change Summary

| File | Action | Description |
|------|--------|-------------|
| `src/utils/types.ts` | MODIFY | Add `NemesisRecord`, `NemesisRank`, `NemesisState` interfaces; add `isNemesis?: boolean` to `TankEntity` |
| `src/utils/constants.ts` | MODIFY | Add `NEMESIS_*` constants block |
| `src/game/NemesisSystem.ts` | **NEW** | Core nemesis logic: persistence, name gen, stat scaling, creation, defeat, roaming, queries |
| `src/data/nemesisMonologues.ts` | **NEW** | 25 monologue lines + getter functions |
| `src/data/npcDialogs.ts` | MODIFY | Add `'nemesis'` to `NPCType`, add `NPC_TYPE_COLORS.nemesis`, add `createNemesisDialog()` factory |
| `src/data/npcAssets.ts` | MODIFY | Add nemesis entry to `NPC_ASSET_URLS` |
| `src/game/GameState.ts` | MODIFY | Add `nemesisState`, `setNemesisState`, `triggerNemesisDialog` to store |
| `src/screens/OpenWorldScreen.tsx` | MODIFY | Nemesis spawning on valley entry, nemesis creation on player death, nemesis defeat handling |
| `src/game/AISystem.ts` | MODIFY | Nemesis-aware AI overrides (detection, retreat, speed, weapons) |
| `src/components/ui/HUD.tsx` | MODIFY | Add nemesis detection indicator |

**Total: 2 new files, 8 modified files**
