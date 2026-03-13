# Co-op Shooter — Evolution Plan

Two players share a keyboard. Currently they feel identical — same bullets, same speed, same everything. The goal: differentiate them into distinct roles (Blue = Elf/Sniper, Red = Dwarf/Tank), add Dynasty Warriors-style enemy swarms, and create a co-op wave / PvP intermission game loop.

Each phase is independently playtestable. Don't skip ahead.

---

## Phase 1: Red Dwarf Shockwave ← START HERE

The most novel mechanic — makes the two characters feel fundamentally different.

### Weapon type system
- Add `weaponType: 'sniper'` to P1 controls, `weaponType: 'slam'` to P2 controls
- Branch in `Player.update()` shooting block (~line 954) based on `weaponType`

### Blue/P1 Sniper (minor tuning)
- Keep existing auto-fire-on-hold behavior
- New CFG params: `sniperFireRate: 8`, `sniperBulletSpeed: 1100` (faster than current 900), `sniperBulletSize: 2.5` (smaller = precision feel), `sniperDamage: 1`
- Reference these instead of generic bullet params

### Red/P2 Ground Slam (new mechanic)
- **Hold shoot → charge**: `chargeTime` accumulates up to `CFG.slamChargeRate` (1.0s)
- **Release → shockwave**: cone-shaped AOE radiates outward from Red in aim direction
- Charge level scales: radius (`slamMinRadius` → `slamMaxRadius`), cone half-angle (`slamMinAngle` → `slamMaxAngle`)
- **Movement penalty**: 50% max speed while charging (vulnerable, needs Blue covering)
- **Charge visual**: growing arc preview around Red showing the cone that will fire, plus a charge ring (arc progress around player)
- **Release effects**: shockwave entity, heavy screen shake (proportional to charge), particle burst in cone direction, recoil push backward

### Shockwave entity system
New `shockwaves[]` array (same pattern as bullets/ghosts). Each shockwave has:
- `x, y, aimAngle, halfAngle` — cone geometry
- `currentRadius` — expanding outward at `CFG.slamSpeed` (600 px/s)
- `maxRadius` — determines lifetime
- `hasHit: new Set()` — prevents double-damage

### Cone collision detection
```
isInCone(sx, sy, tx, ty, aimAngle, halfAngle, innerR, outerR)
```
- Distance check: target must be within the expanding ring band (~40px wide)
- Angle check: `atan2` angle difference must be within `halfAngle`
- The ring expands outward, hitting entities as it passes through them (wave effect)

### Shockwave rendering
- Semi-transparent filled arc (the ring band) fading as it expands
- Stroke on the outer arc edge
- Alpha decreases with progress

### New CFG params
```
sniperFireRate: 8, sniperBulletSpeed: 1100, sniperBulletSize: 2.5, sniperDamage: 1
slamChargeRate: 1.0, slamMinRadius: 60, slamMaxRadius: 200
slamMinAngle: 0.4, slamMaxAngle: 1.2
slamSpeed: 600, slamDamage: 30, slamKnockback: 400
slamCooldown: 0.3, slamChargeSlowdown: 0.5
```

### New debug panel section: "Slam"
Sliders for all slam params + sniper params.

### Playtest checklist
- [ ] Red feels totally different from Blue — charge-and-release timing vs continuous fire
- [ ] Charging slows you, creating tactical vulnerability
- [ ] Cone aiming with 8-dir keys = satisfying area denial
- [ ] Debug sliders for instant tuning

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

## Phase 5: Polish (later)
- Enemy variety: fast scouts, slow brutes
- Kill streak feedback (10+ kills in one slam → big shake + flash)
- Spawn warning indicators at arena edges
- Arena shrink between waves
- Web Audio oscillator sounds (no assets)
- Camera improvements (zoom consideration for enemy positions)

---

## Verification (Phase 1)
1. Open in browser, start playing
2. P1 (Blue): hold Left Option → auto-fires bullets (faster, smaller than before)
3. P2 (Red): hold Right Option → charge indicator grows around player
4. P2: release Right Option → shockwave cone expands outward in aim direction
5. Longer hold = bigger cone radius and wider angle
6. P2 moves slower while charging
7. Shockwave damages P1 (friendly fire still on in current mode)
8. All slam params tuneable via debug panel "Slam" section
9. Keyboard overlay updated: P2 shoot label says "slam" instead of "shoot"
