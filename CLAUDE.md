# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Two-player same-keyboard co-op top-down shooter. Priority is incredible movement feel (Super Meat Boy snappiness). Currently a single-file vanilla JS game (`index.html`) — no build step, no dependencies.

## Running

Open `index.html` directly in a browser. No server or build required.

## Architecture

Everything lives in a single `index.html` with inline `<style>` and `<script>`. The JS is organized into labeled sections:

- **CFG** — Global config object. Every gameplay parameter lives here and is exposed via the debug panel. When adding new mechanics, add their tuning parameters to CFG.
- **Input System** — Tracks `keys` map and `mouse` position. Prevents default on Alt/Option keys to suppress special character insertion. Space is edge-triggered (not held) for the swap mechanic.
- **Gamepad System** — Polls `navigator.getGamepads()` each frame. Provides deadzone handling, edge-triggered button detection via `prevGamepadButtons`, and helper functions (`getGamepad`, `applyDeadzone`, `gamepadButtonJustPressed`). `gamepadIndex` lives on the Player object (not in `controls`) so gamepads stay with the player body during control swaps.
- **Camera** — Follows midpoint between players, zooms out based on inter-player distance. Provides `worldToScreen`/`screenToWorld` coordinate conversion and screen shake. All rendering between `applyTransform()`/`restoreTransform()` is in world space.
- **Player** (class) — Movement, aiming, shooting, dashing, collision. Controls are config-driven via a `controls` object that specifies key bindings and aim mode (`'mouse'` vs `'keys'`). Controls can be swapped between players at runtime.
- **Swap System** — `swapControls()` swaps control configs and positions between players. Leaves ghost afterimages, spawns particle trails between origin/destination, grants i-frames, and updates the keyboard overlay colors dynamically.
- **Bullets/Particles/Ghosts** — Simple array-based entity pools. Entities have a `life` timer and are spliced out when expired. Bullets check collision against obstacles and players (skipping owner + i-framed targets).
- **Obstacles** — Static rectangles defined as proportions of arena size in `generateObstacles()`.
- **Controls Modal** — CSS-rendered MacBook Pro keyboard diagram showing P1/P2 key assignments. Colors update dynamically on swap. Toggle with `Escape` key.
- **Debug Panel** — Built dynamically from a section/param config in `buildDebugPanel()`. Sliders mutate CFG directly. Toggle with `?` key.

## Controls

| | P1 (cyan) | P2 (coral) |
|---|---|---|
| Move | WASD | IJKL |
| Aim | Mouse (analog 360°) | Arrow keys (8-dir) |
| Shoot | Left Option | Right Option |
| Dash | Left Shift | Right Shift |
| Swap | Space (shared) | Space (shared) |

### Gamepad (Xbox / Standard layout)

| Input | Action |
|---|---|
| Left Stick | Move (analog) |
| D-Pad | Move (digital) |
| Right Stick | Aim (analog 360°) |
| RT (R2) | Shoot |
| LB (L1) | Dash |
| A (Cross) | Swap |
| Start | Respawn |

Gamepad 0 → P1, Gamepad 1 → P2. Both players get full analog aim via right stick. Keyboard input still works as fallback when gamepad stick/d-pad is idle.

`Space` swaps controls AND positions between players (either player can trigger it unilaterally). `R` respawns dead players. `?` toggles debug panel. `Escape` toggles keyboard controls diagram.

## Key Design Decisions

- **Asymmetric aim:** P1 gets analog mouse aim with a dashed line-of-sight line extending to the arena edge. P2 gets 8-direction key aim with a shorter directional indicator. This is intentional — lean into the asymmetry, don't try to unify them.
- **Movement model:** High acceleration + exponential friction = near-instant response. Friction is stronger when no input is held (snappy stop) vs while moving (more control). Tune via debug panel, not by changing the model.
- **Dash is a teleport blink**, not a fast-move. Leaves a ghost afterimage at origin, grants i-frames. If no movement input, dashes in aim direction.
- **Swap is a dramatic teleport exchange.** Both players swap positions and controls simultaneously. Ghosts at origin, particle trails along the path, burst at arrival, heavy screen shake, 0.3s i-frames. Only works when both players are alive. Cooldown planned but not yet implemented.
- **Friendly fire is always on.** Future PvP toggle is planned but the collision system assumes bullets can hit any non-owner player.
- **All visual feedback matters:** screen shake, muzzle particles, hit flash, movement trails, dash ghosts, bullet glow. Don't remove these — they're core to the feel.

## Adding New Debug Parameters

1. Add the value to `CFG`
2. Add a slider entry to the appropriate section in `buildDebugPanel()`
3. Reference `CFG.yourParam` in game code — it updates live

## Coordinate System

- World space: (0,0) is top-left of arena, extends to (arenaWidth, arenaHeight)
- The camera transforms world→screen. Mouse input must be converted via `camera.screenToWorld()` for gameplay use.
