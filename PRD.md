# 1P Mode + Gamepad Fix — PRD

## Overview

Add a single-player mode where Blue (P1) fights a CPU-controlled Red opponent, plus fix gamepad input so Bluetooth controllers fully override keyboard/mouse.

## Start Screen

- Title card shown on load: game title, brief controls hint, mode selection
- Two options: "1 Player" and "2 Players"
- Minimal, terminal-aesthetic design consistent with existing UI (JetBrains Mono, dark bg)
- Selecting a mode starts the game; no further menus

## CPU Red Player (1P Mode)

### Behavior
- Actively engages Blue: moves toward player, shoots when in range, uses dashes tactically
- Uses the shotgun weapon (its default) with all existing effects
- Occasionally charges and uses the 360° blast wave special weapon when Blue is nearby

### Navigation
- Hybrid raycast + unstick: cast rays toward Blue, move toward player
- If stuck (no position change for >0.5s), pick a random perpendicular direction to escape
- Respects arena bounds and obstacle collision (uses existing Player physics)

### Adaptive Difficulty
- Tracks Blue's kill count and current HP
- **Difficulty signals:**
  - If Blue is dominating (high kill ratio, high HP): CPU aims more accurately, moves faster, dashes more often, uses specials more
  - If Blue is struggling (low HP, few kills): CPU aims less accurately, slower reaction times, less aggressive
- Difficulty adjusts smoothly, not in sudden jumps

### Respawn
- CPU auto-respawns ~3 seconds after death at its spawn point
- Brief countdown shown at death position
- Respawn includes i-frames (same as current respawn)

### Swap
- Swap mechanic (Space) is disabled in 1P mode

## 1P HUD
- Small kill counter in the corner
- Respawn countdown displayed at CPU's death position

## Gamepad Input Fix

- When `player.gamepadIndex` is assigned AND a gamepad is actually connected:
  - **All** keyboard input for that player is ignored (movement keys, shoot key, dash key)
  - Mouse aim is ignored for that player (even if aimMode is 'mouse')
  - Only gamepad axes and buttons control that player
- Detection: check `getGamepad(player.gamepadIndex)` is non-null each frame
- This prevents the trackpad/mouse from interfering with gamepad aim on Bluetooth controllers
