# Co-op Shooter — Evolution Plan

Two players share a keyboard. Currently they feel identical — same bullets, same speed, same everything. The goal: differentiate them into distinct roles (Blue = Elf/Sniper, Red = Dwarf/Tank), add Dynasty Warriors-style enemy swarms, and create a co-op wave / PvP intermission game loop.

Each phase is independently playtestable. Don't skip ahead.

---

## Phase 0: Game Juice & Feel ← START HERE

Make every existing action feel incredible before adding new mechanics. These are layered feedback systems — each one is small, but stacked together they transform the game from "functional" to "visceral."

### Chunk 0A: Hit Feedback (3 features)

The most impactful juice pass. Every hit should feel devastating.

**Hitstop / freeze frames**
- On bullet hit, pause the game loop for 50-80ms (2-3 frames at 60fps)
- Implementation: set `hitstopTimer` in game state; skip `update()` while timer > 0, keep rendering
- Scale duration with damage: light hits 40ms, heavy hits 80ms
- New CFG params: `hitstopDuration: 0.05`, `hitstopHeavy: 0.08`
- This single feature is what makes Hollow Knight and Nuclear Throne hits feel weighty

**Impact sparks / hit markers**
- On bullet→player or bullet→obstacle collision, spawn a bright white cross/star shape at impact point
- Lasts ~3 frames (50ms), stationary (unlike particles which drift)
- Implementation: new `sparks[]` array, each spark has `x, y, life, size`
- Render as 4 short lines radiating from center (cross pattern), bright white, hard alpha cutoff (no fade)
- Different from existing particles — these are crisp, geometric, immediate

**Damage screen flash / vignette**
- On player taking damage, flash a colored vignette overlay at screen edges
- Radial gradient from transparent center → semi-transparent color at edges
- Color matches the shooter's color (hit by Blue = cyan flash, hit by Red = coral flash)
- Duration: 100ms fade-in, 200ms fade-out
- Render in screen space (after `restoreTransform()`), not world space
- New CFG param: `damageFlashAlpha: 0.3`

#### Playtest checklist
- [ ] Hits feel weighty — the brief pause makes each one land
- [ ] Sparks give instant visual confirmation of where the hit landed
- [ ] Screen flash communicates "you're hurt" through the whole screen, not just the character
- [ ] None of these interfere with gameplay readability

---

### Chunk 0B: Shooting Feel (2 features)

Make firing weapons feel powerful through camera and visual feedback.

**Directional camera punch**
- On weapon fire, offset camera briefly in the direction *opposite* the shot
- Different from screen shake (random jitter) — this is a deliberate directional kick
- Implementation: add `camera.punchX`, `camera.punchY` that decay back to 0 via lerp
- Punch magnitude: sniper 3px, shotgun 8px, bomb 12px (scaled by charge)
- Decay: fast snap-back, ~100ms to return (lerp factor ~0.15)
- Apply punch offset in `applyTransform()` alongside shake offset
- New CFG params: `cameraPunchSniper: 3`, `cameraPunchShotgun: 8`, `cameraPunchBomb: 12`, `cameraPunchDecay: 0.15`

**Muzzle bloom**
- On weapon fire, draw a brief bright circle at the muzzle point (player edge in aim direction)
- White core → weapon color → transparent, fading over 50ms
- Radius: sniper 6px, shotgun 12px
- Implementation: add to existing `spawnDirectionalParticles` call sites, or new `muzzleFlashes[]` array
- Render with additive-style blending (globalCompositeOperation = 'lighter') for glow effect
- Pairs with existing directional particles but adds the "energy source" glow they're missing

#### Playtest checklist
- [ ] Shooting feels like it has physical force — the camera kicks with each shot
- [ ] Shotgun kick is noticeably stronger than sniper
- [ ] Muzzle bloom gives each weapon a distinct firing signature
- [ ] Camera punch doesn't cause motion sickness at high fire rates (test sniper at 10/s)

---

### Chunk 0C: Death & Environmental Feedback (2 features)

Make consequences visible and persistent in the world.

**Death explosion / disintegration**
- Replace instant disappearance with dramatic shatter effect
- On death: spawn 12-16 arc-segment particles that look like pieces of the player's body
  - Each segment: small arc (30° chunk of the player circle), flies outward with rotation
  - Color matches player, fades over 0.8s
  - Initial velocity: random outward at 200-400 px/s
- Brief white screen flash (2 frames, low alpha 0.15) on death — screen-space overlay
- Optional: 80ms hitstop on death hit (heavy hitstop)
- Brief slow-motion: set `gameSpeed = 0.3` for 200ms after death, lerp back to 1.0
  - Implementation: multiply `dt` by `gameSpeed` in the update loop
  - New CFG params: `deathSlowmo: 0.3`, `deathSlowmoDuration: 0.2`

**Arena impact decals**
- When bullets hit obstacles, leave a small dark scorch mark (3-4px circle)
- Semi-transparent, fades over 8-10 seconds
- New `decals[]` array, each: `{ x, y, size, life, maxLife, color }`
- Cap at 60 decals (remove oldest when pool full)
- Render beneath everything else (first thing after arena background)
- After a firefight, the arena tells the story of the battle
- New CFG params: `decalLifetime: 8`, `decalMaxCount: 60`

#### Playtest checklist
- [ ] Deaths feel dramatic and cinematic — the brief slowmo creates a "moment"
- [ ] Player shattering into pieces is visually clear (you can tell what happened)
- [ ] Decals accumulate naturally during gameplay without performance impact
- [ ] Decal cap prevents unbounded growth

---

## Phase 1: Weapon Differentiation

Make the two players feel fundamentally different. Primary weapons stay as-is (sniper + shotgun), but each player gets a **special weapon** activated by double-tap or hold on their shoot key.

### Chunk 1A: Special Weapon Trigger System (2 features)

**Double-tap / hold detection**
- Detect two input modes on the shoot key: tap (normal fire) vs hold/double-tap (special weapon)
- **Hold mode**: if shoot key held > `CFG.specialChargeThreshold` (0.3s), switch to special weapon charging
- Normal tap-and-release fires primary weapon as usual
- Implementation: track `shootHoldTime` per player; if it crosses threshold, set `chargingSpecial = true`
- While charging special, suppress primary fire
- New CFG param: `specialChargeThreshold: 0.3`

**Weapon type system**
- Add `specialWeapon: 'laser'` to P1 controls, `specialWeapon: 'blastwave'` to P2 controls
- Branch in `Player.update()` based on `chargingSpecial` + `specialWeapon`
- Charge state: `chargeTime` accumulates while held, `chargeLevel = chargeTime / maxChargeTime`
- Release triggers the special weapon
- Movement penalty while charging: `CFG.specialChargeSlowdown: 0.5` (50% max speed)

#### Playtest checklist
- [ ] Tap fires normally, hold triggers special — feels natural, no accidental misfires
- [ ] 0.3s threshold is long enough to avoid false triggers during rapid fire
- [ ] Movement slowdown while charging creates vulnerability

---

### Chunk 1B: Blue Laser Beam (3 features)

**Laser beam mechanics**
- Hold special → charge, release → fires a continuous beam in aim direction
- Beam is a line from player to arena edge (or first obstacle hit)
- Duration: scales with charge level (0.3s min → 1.0s max)
- Damage: continuous tick damage to anything touching the beam (every 0.1s)
- Width: thin (3px core, 8px glow)
- New CFG params: `laserMinDuration: 0.3`, `laserMaxDuration: 1.0`, `laserDamage: 8`, `laserTickRate: 0.1`, `laserWidth: 3`

**Laser recoil-as-movement**
- While beam is active, player is pushed backward (opposite aim direction) continuously
- Recoil force strong enough to slide the player across the arena — the laser is a movement tool
- Players can aim the laser to propel themselves (like Luftrausers' shoot-to-move)
- Recoil strength: `CFG.laserRecoilForce: 600` (px/s² acceleration applied opposite aim)
- This makes positioning during laser fire a skill — you're sliding backward while trying to aim

**Laser rendering**
- Core: bright white line (3px), full opacity
- Inner glow: weapon-colored line (8px), 50% opacity
- Outer glow: weapon-colored line (16px), 15% opacity, composite 'lighter'
- Flickering: slight random width variation each frame (±1px on glow layers)
- Impact point: bright spark + small particle burst where beam hits obstacle/arena edge
- Charge preview: thin dashed line in aim direction while charging (like existing LOS but brighter)

#### Playtest checklist
- [ ] Laser feels powerful — a commitment ability with high reward
- [ ] Recoil sliding is fun to control, not frustrating
- [ ] Beam visually reads clearly against the dark arena
- [ ] Charging laser while being chased = tense risk/reward moment

---

### Chunk 1C: Red Blast Wave (3 features)

**Shotgun / blast wave mechanics**
- Hold special → charge, release → fires a cone-shaped shockwave from Red in aim direction
- Charge level scales: radius (`slamMinRadius: 60` → `slamMaxRadius: 200`), cone half-angle (`slamMinAngle: 0.4` → `slamMaxAngle: 1.2`)
- Expanding ring band (~40px wide) radiates outward at `CFG.slamSpeed: 600` px/s
- `hasHit: new Set()` prevents double-damage on same target
- New CFG params: same as original Phase 1 slam params

**Blast wave moves obstacles**
- Shockwave is strong enough to push obstacle rectangles
- On shockwave hitting an obstacle: apply force vector (from blast origin toward obstacle center)
- Obstacles get temporary velocity, decelerate via friction, then stop at new position
- Implementation: add `vx, vy` to obstacle objects (normally 0); in shockwave collision, set velocity proportional to `CFG.slamKnockback`
- Obstacle-obstacle collision: simple push-apart (prevent overlap)
- Obstacle-arena-edge collision: clamp to arena bounds
- This fundamentally changes arena layout mid-game — Red reshapes the battlefield
- New CFG param: `obstacleMoveFriction: 0.95`, `obstacleMass: 3` (heavier = harder to push)

**Blast wave rendering**
- Semi-transparent filled arc (the ring band) fading as it expands
- Stroke on the outer arc edge
- Inner shockwave distortion: briefly push nearby particles outward (if any exist in the cone)
- Charge preview: growing arc around Red showing the cone that will fire
- Screen shake proportional to charge level

#### Playtest checklist
- [ ] Blast wave feels like a powerful area denial tool
- [ ] Moving obstacles is surprising and satisfying the first time it happens
- [ ] Arena reshaping creates emergent tactical situations
- [ ] Obstacle movement feels physical (weight, friction, not instant teleport)

---

### Chunk 1D: Big Bomb / Expanding Ring (2 features)

**Expanding bomb ring** (evolution of existing bomb mechanic)
- Full 360° expanding ring from player position (Red's ground-pound variant)
- Larger radius than directional blast wave but less focused damage
- Knocks back everything in all directions — players, future enemies, obstacles
- Higher charge requirement: must hold for full charge (1.5s) to unlock bomb vs blast wave
  - Partial charge (< 1.0s) = directional blast wave
  - Full charge (≥ 1.0s) = 360° expanding ring
- Visual distinction: charge indicator shifts from arc preview to full ring preview at threshold
- New CFG params: `bombFullChargeTime: 1.5`, `bombRadius: 300`, `bombDamage: 20`, `bombKnockback: 500`

**Bomb environmental interaction**
- Ring pushes all obstacles outward from center (radial force)
- Heavier push than directional blast wave (bomb is the "room clearer")
- Leaves decal ring on arena floor (fades over 5s) — visual evidence of the explosion
- Massive screen shake + hitstop on detonation

#### Playtest checklist
- [ ] Holding longer for bomb vs releasing early for blast wave feels like a meaningful choice
- [ ] 360° ring reads differently from directional cone visually
- [ ] Environmental destruction from bomb is dramatic
- [ ] Charge level indicator clearly communicates blast wave → bomb transition

---

## Phase 2: Enemy Swarm System

### Performance architecture (critical)
Current splice-based arrays won't scale to 200 enemies.

**Object pool**: Pre-allocated array + active flags. `acquire()` returns an inactive slot, `release(idx)` marks it inactive. No GC pressure. Pools: enemies (300), optionally migrate bullets (200) and particles (500).

**Spatial hash grid**: 100px cells across 5000×5000 arena (50×50 = 2500 cells). Each frame: clear → insert all enemies → bullets query nearby cells instead of iterating all enemies. Reduces collision from O(n×m) to O(n×k) where k ≈ 5.

### Enemy entity (plain object from pool, not class)
- `x, y, vx, vy, hp, size, speed, hitFlash, retargetTimer, targetPlayer, targetOffsetX/Y`
- Simple visual: small colored circle (size ~8px), stroke + transparent fill
- Batch rendering: single fill color, minimal state changes

### Enemy AI
- Pick nearest alive player as target, recalculate every 0.3–0.8s (staggered to spread load)
- Move toward target with slight random offset (swarm feel, not single-file)
- Contact damage on touching a player (enemy dies on contact)

### Collision integration
- **Bullets vs enemies**: In `updateBullets`, query spatial grid near bullet, distance check, apply `sniperDamage`
- **Shockwaves vs enemies**: In `updateShockwaves`, query grid with `outerR` radius, filter with `isInCone` — this is where Red shines (clearing groups)
- **Enemies vs players**: In `updateEnemies`, check proximity to each player

### Wave spawner
- Enemies spawn at random arena edges
- `enemiesPerWave = baseCount + waveNumber * scaling`
- Spawn interval configurable (0.15s default — fast stream)

### New CFG params
```
enemyHP: 2, enemySize: 8
enemyMinSpeed: 80, enemyMaxSpeed: 150
enemyContactDamage: 15, enemySpawnInterval: 0.15
baseEnemiesPerWave: 20, enemiesPerWaveScaling: 10
```

### New debug panel section: "Enemies"

### Playtest checklist
- [ ] Dynasty Warriors fantasy: Blue snipes from range, Red wades in and slams
- [ ] Enemy HP tuning (1 sniper hit? 2? Slam clears groups?)
- [ ] Performance at 100, 150, 200 enemies
- [ ] Entity count visible in debug HUD

---

## Phase 3: Wave/PvP Game Loop

### State machine
```
WAVE_COUNTDOWN → COOP_WAVE → PVP_INTERMISSION → WAVE_COUNTDOWN → ...
```

### Co-op wave
- Enemies spawn, players fight together
- No friendly fire (bullets pass through other player)
- Track kills per player (score)

### PvP intermission
- Triggers when all enemies are dead
- Both players respawn at full HP, repositioned center
- Friendly fire on, weapons turn on each other
- Timer-based (`CFG.pvpDuration: 15s`) or until one player dies
- Winner gets bonus/credit (future)

### HUD
- Minimal during co-op: wave number + enemies remaining (top-left, subtle)
- "DUEL" indicator during PvP with countdown timer
- Wave countdown between phases

### New CFG params
```
pvpDuration: 15, waveCountdown: 3, friendlyFireInCoop: false
```

### Playtest checklist
- [ ] The core interesting mechanic: cooperation → competition → cooperation
- [ ] Emotional whiplash of fighting together then fighting each other
- [ ] PvP balance: sniper (range) vs slam (area)

---

## Phase 4: Swap Mechanic Evolution (later)
- Space swap also swaps weapon types + colors (identity follows weapon)
- Reset charge state on swap
- Swap cooldown (`CFG.swapCooldown: 1.0`)
- Optional "shared mode" (one moves, one aims) — bigger design change, defer until core loop is proven

## Phase 5: Further Polish (later)
- Enemy variety: fast scouts, slow brutes
- Kill streak feedback (10+ kills in one slam → big shake + flash)
- Spawn warning indicators at arena edges
- Arena shrink between waves
- Web Audio oscillator sounds (no assets)
- Camera improvements (zoom consideration for enemy positions)

---

## Implementation Chunk Summary

| Chunk | Features | Size | Depends on |
|-------|----------|------|------------|
| **0A** | Hitstop, impact sparks, damage vignette | Small (3) | None |
| **0B** | Camera punch, muzzle bloom | Small (2) | None |
| **0C** | Death explosion, arena decals | Small (2) | 0A (hitstop/slowmo) |
| **1A** | Special weapon trigger, weapon type system | Medium (2) | None |
| **1B** | Laser beam, recoil movement, laser rendering | Medium (3) | 1A |
| **1C** | Blast wave, moveable obstacles, blast rendering | Medium (3) | 1A |
| **1D** | Expanding bomb ring, bomb environment | Medium (2) | 1C (blast wave) |
| **2** | Enemy swarm system | Large | 0A-0C, 1A-1D |
| **3** | Wave/PvP game loop | Large | 2 |

Chunks 0A and 0B can be done in parallel. 1B and 1C can be done in parallel after 1A.

---

## Verification (Phase 0)
1. Open in browser, shoot the other player
2. Brief freeze on hit (hitstop) — game pauses for a beat
3. White cross spark appears at impact point
4. Screen edges flash with shooter's color
5. Fire sniper: camera kicks opposite aim direction, small muzzle glow
6. Fire shotgun: camera kicks harder, bigger muzzle glow
7. Kill a player: body shatters into arc fragments, brief slowmo, white flash
8. Shoot obstacles: small dark scorch marks accumulate on surfaces
9. All new params tuneable via debug panel

## Verification (Phase 1)
1. P1 (Blue): tap Left Option → normal sniper fire
2. P1: hold Left Option > 0.3s → starts charging laser
3. P1: release → laser beam fires, pushing player backward
4. P2 (Red): tap Right Option → normal shotgun fire
5. P2: hold Right Option < 1.0s → charges blast wave (cone preview)
6. P2: release → directional shockwave, pushes obstacles
7. P2: hold Right Option ≥ 1.5s → charges 360° bomb (full ring preview)
8. P2: release → expanding ring, pushes everything outward
9. All special weapon params tuneable via debug panel
