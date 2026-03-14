# TODO — 1P Mode + Gamepad Fix

## Gamepad Input Fix
- [ ] Add `hasActiveGamepad(player)` helper that checks if gamepad is connected
- [ ] In `getInput()`: skip keyboard when gamepad active
- [ ] In `getAimDirection()`: skip mouse/key aim when gamepad active
- [ ] In `isShooting()` / `isDashing()`: skip keyboard when gamepad active

## Start Screen
- [ ] Add title card HTML/CSS overlay
- [ ] "1 Player" / "2 Players" mode selection
- [ ] Wire mode selection to game state; hide overlay on start
- [ ] Show controls hint for selected mode

## CPU AI System
- [ ] Create `CPUController` class that produces input signals (movement, aim, shoot, dash)
- [ ] Integrate with Player — when CPU-controlled, use CPUController instead of keyboard/gamepad
- [ ] Raycast-based targeting: aim toward Blue
- [ ] Hybrid navigation: move toward Blue, unstick if stuck for >0.5s
- [ ] Shooting logic: fire when aimed at Blue within range
- [ ] Dash logic: dash toward/away tactically
- [ ] Special weapon: occasionally charge 360° blast when Blue is close

## Adaptive Difficulty
- [ ] Track Blue kills and HP
- [ ] Compute difficulty factor (0–1) from kill ratio + HP
- [ ] Scale CPU accuracy, reaction delay, aggression, dash frequency, special usage

## 1P Mode Integration
- [ ] Disable swap mechanic in 1P
- [ ] CPU auto-respawn after ~3s with countdown at death position
- [ ] Kill counter HUD element
- [ ] Hide P2 keyboard controls from controls modal in 1P

## Polish
- [ ] Ensure all existing effects work with CPU (particles, shake, etc.)
- [ ] Debug panel: add CPU difficulty section for tuning
