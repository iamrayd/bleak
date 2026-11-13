# Bleak Plan 07 — UI, Input, Lobby and Meta

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A player can understand and control everything with keyboard and mouse, gamepad or touch, with a HUD that stays out of the way.

**Architecture:** Controllers read replicated attributes and send requests. Screens are built in code from one theme module. Input goes through `InputController` only. UI never decides a game rule: it shows what `PromptRules` and the server say.

**Tech Stack:** ScreenGui, UserInputService and ContextActionService behind `InputController`, TweenService, GuiService.

**Spec:** `docs/spec/2026-11-13-bleak-game-design.md` sections 4, 5, 6, 7, 13, 15, 17.

Agent: **UI**. Every task uses the standard task cycle (master plan
section 7). Visual checks are T3 screenshots at 1920 × 1080, 1366 × 768 and a
phone size (844 × 390) using Studio's device emulator (ask the user to switch
device if it cannot be done by MCP).

UI rules for every task:

- Diegetic first: in play the HUD shows only stamina, lamp charge, held item, the prompt, and teammates' state.
- Text ≥ 16 px at 1080p, scaled by `UIScale` from the viewport height. Contrast ≥ 4.5 : 1.
- Every interactive element works with mouse, gamepad focus and touch. Touch targets ≥ 44 px.
- No element sits under the top-bar inset or in the bottom-corner thumb zones on touch.
- Every string comes from `Strings.luau`.
- Transitions ≤ 0.25 s. No blocking animations.

---

### Task U1: InputController

**Files:**
- Create: `src/client/Controllers/InputController.luau`, `src/shared/Config/InputConfig.luau` (Foundation approves the new config file)
- Create: `src/shared/Logic/InputMap.luau`
- Test: `src/tests/unit/InputMap.spec.luau`

**Interfaces — Produces:**

```lua
-- Actions: "Sprint", "Crouch", "Interact", "Lamp", "Beam", "Drop", "Use", "Menu",
--          "SpectateNext", "SpectatePrev", "Ready"
InputController.actionChanged: Signal<string, boolean>       -- action, isDown
InputController.isDown(action: string): boolean
InputController.device(): "KeyboardMouse" | "Gamepad" | "Touch"
InputController.deviceChanged: Signal<string>
InputController.setEnabled(group: "Gameplay" | "Menu", enabled: boolean)
InputController.glyph(action: string): string                -- text or image id for prompts
InputMap.resolve(bindings, device: string, input: string): string?     -- input name -> action
InputMap.conflicts(bindings): { { device: string, input: string, actions: {string} } }
```

Default bindings:

| Action | Keyboard / mouse | Gamepad | Touch |
|--------|------------------|---------|-------|
| Move, look | WASD, mouse | Left stick, right stick | Roblox thumbstick, drag |
| Sprint | LeftShift (hold) | L3 (toggle) | Button (toggle) |
| Crouch | C or LeftControl (toggle) | B (toggle) | Button (toggle) |
| Interact | E (press or hold) | X | Context button |
| Lamp | F | Y | Button |
| Beam | Right mouse (hold) | L2 (hold) | Button (hold) |
| Drop | G | DPad Down | Button |
| Use | Left mouse | R2 | Button |
| Menu | Tab or Escape passthrough | Start | Button |
| Spectate next / prev | D / A or mouse buttons | R1 / L1 | Arrows |
| Ready | R | X in lobby | Button |

**Tests:** `resolve(defaults, "KeyboardMouse", "E") == "Interact"`; `("Gamepad", "ButtonX") == "Interact"`; an unbound key → nil; `conflicts(defaults)` is empty; adding a second action on `E` reports one conflict.

**Behaviour:** `Gameplay` is disabled while a menu is open, in `Lobby` UI focus, and when the chat box is focused. The last device used decides `device()`. Jump is unbound everywhere (`ContextActionService` sink for the jump action).

- [ ] Follow the standard task cycle.
- [ ] Commit: `feat: add input controller`.

---

### Task U2: Theme and HUD shell

**Files:**
- Create: `src/client/UI/Theme.luau`, `src/client/UI/Components.luau`, `src/client/UI/Hud.luau`, `src/client/Controllers/HudController.luau`

**Theme:** background near-black blue (12, 16, 20) at 70% opacity; text off-white (226, 230, 232); accent lamp-amber (255, 184, 92); danger ember (232, 96, 60); one serif display font for titles and one sans font for body, both from the Roblox font list. Corners 4 px. No drop shadows; a 1 px inner stroke.

**Components:** `Components.label`, `button`, `bar`, `ring`, `panel`, `list`, `toggle`, `slider`, each taking a props table and returning an instance plus an `update(props)` function. `button` handles hover, press, gamepad selection and `Ui_Hover` / `Ui_Click` cues.

**HUD elements (bottom centre unless noted):**
- Stamina bar: hidden at 100; appears when below 100; turns ember when `Exhausted`.
- Lamp charge: a small arc by the crosshair dot; ember when `LampLocked`; hidden on Nightmare.
- Held item icon and name (bottom right). Candle shows remaining seconds as a shrinking ring.
- Crosshair: a 3 px dot, 50% opacity.
- Latch warning: the screen effect is VfxController's. The HUD adds the text `Strings.BrinelingOnYou` while `LatchMeter` > 0. There is no shake-off mechanic: the player escapes by moving away or by a beam.
- Last Breath pips (solo only), top left.

- [ ] Follow the standard task cycle. T3: drive each attribute with DebugService and screenshot each state.
- [ ] Commit: `feat: add theme and HUD`.

---

### Task U3: Interaction prompts

**Needs:** C6 (`PromptRules`).

**Files:** Create `src/client/Controllers/InteractionController.luau`, `src/client/UI/Prompt.luau`. Create `src/shared/Logic/TargetPick.luau`. Test: `src/tests/unit/TargetPick.spec.luau`.

```lua
-- candidates: things within range on screen
TargetPick.best(candidates: { { id: any, distance: number, angleDeg: number, hasPrompt: boolean } },
  maxDistance: number, maxAngle: number): any?
-- Picks the candidate with a prompt, within limits, with the smallest angle; ties by distance.
```

**Behaviour:**
- Every frame: raycast from the camera centre up to 7 studs. If it hits something registered (item, door, cage, bowl, sconce, downed character), that is the target. If not, use `TargetPick.best` over registered things within 7 studs and 12° of the centre (aim assist; 20° on gamepad and touch).
- Build `TargetInfo` and `ActorInfo` from attributes and tags, call `PromptRules.resolve`, and show the prompt: glyph, label, and for holds a ring.
- A locked door with no key shows the hint "Locked — needs the <key name>" with the key's tag shape icon, no glyph.
- Press: send `Interact(target, "Begin")`. For holds, send `"Cancel"` on release. The ring is driven by the server's `HoldProgress` (start time and duration), not by local timing.
- Highlight: one `Highlight` instance reused on the current target, outline only, amber, 40% opacity.
- `Notify` codes show a short line under the prompt for 2 s (`Busy`, `Full`, `NoFuel`, `NoWisp`, `NotLit`, `Locked`).

**Tests:** `best` returns nil for an empty list; ignores candidates without a prompt; prefers 3° at 6 studs over 8° at 2 studs; ignores a candidate at 13° when the limit is 12; ties on angle pick the nearer.

- [ ] Follow the standard task cycle.
- [ ] Commit: `feat: add interaction prompts`.

---

### Task U4: Lamp and inventory controllers

**Files:** Create `src/client/Controllers/LampController.luau`, `InventoryController.luau`.

- LampController: `Lamp` action → `SetLamp(not LampOn)`. `Beam` down/up → `SetBeam`. While the lamp is on, sends `AimUpdate(camera look vector)` at 15 Hz, and at once when the look direction changes by more than 5°. Stops sending when the lamp is off.
- InventoryController: `Drop` → `DropItem`. `Use` → `UseItem`. Shows a one-line hint the first time each item kind is held (`Strings.Hint_<Kind>`), once per session.
- Both ignore input unless `Status == "Alive"`.

- [ ] Follow the standard task cycle. T3: toggle the lamp, hold the beam until lockout, pick up, swap, drop.
- [ ] Commit: `feat: add lamp and inventory controllers`.

**End of M1 work for UI.**

---

### Task U5: Lobby

**Files:** Create `src/client/Controllers/LobbyController.luau`, `src/client/UI/Lobby.luau`.

- Shown in phase `Lobby` for players with status `Lobby`. Left: player list (name, Ready mark, host mark, rank). Centre: title "BLEAK", a one-paragraph premise, difficulty selector (host only; others see the choice), a short description of the chosen difficulty from `Strings`. Bottom: Ready button; Start button for the host, enabled when everyone is ready.
- Buttons: Perks (U13), Settings (U10), How to Play (five lines: free ten souls; candles lead them; oil feeds the brazier; the beam stuns; revive the fallen).
- `Intro`: a 5-second countdown from `PhaseEndsAt`, then the lobby UI fades out.
- A player who joined mid-match sees "Match in progress — you will join the next one" over the spectate view.

- [ ] Follow the standard task cycle. T3: ready, change difficulty, start solo.
- [ ] Commit: `feat: add lobby UI`.

---

### Task U6: Objective and notifications

**Files:** Create `src/client/UI/Objective.luau`, `src/client/UI/Toasts.luau`.

- Objective (top centre, small): ten soul marks, filled by `Released`. Next to it, Brazier fuel as up to three flame pips, shown only within 30 studs of the Brazier or for 4 s after a change.
- The objective text changes with state: "Find oil and light the Brazier" (unlit) → "Open a cage" (no cage opened) → "Lead a soul to the Brazier" → "Free the souls: n / 10".
- Toasts (top centre, under the objective, max 2 visible): door unlocked (zone name), cage opened, soul freed, teammate downed, teammate carried, teammate revived, Last Breath used, host changed.
- Stage change: no number is shown. The ten-mark row pulses and `Sting_StageUp` plays.

- [ ] Follow the standard task cycle.
- [ ] Commit: `feat: add objective and notifications`.

---

### Task U7: Downed, carried and spectating

**Files:** Create `src/client/Controllers/SpectateController.luau`, `src/client/UI/Downed.luau`, `src/client/UI/Spectate.luau`, `src/client/UI/Teammates.luau`.

- Teammates: a `Highlight` (through walls) on `Downed` teammates and on the carry proxy of `Carried` teammates, ember colour; plus a small edge-of-screen arrow with distance. Alive teammates show a faint name tag within 30 studs only.
- Downed screen: desaturated view, text "You are down. A teammate with a field dressing can help you up." In solo with a pending Last Breath: a 15 s ring.
- Carried: black fade, then the proxy camera (plan 03, P4) with the text "The Keeper has you."
- Spectating (late joiners): camera follows an alive player in third person at 10 studs; `SpectateNext` / `SpectatePrev` cycle; shows the watched player's name.

- [ ] Follow the standard task cycle.
- [ ] Commit: `feat: add downed, carried and spectate UI`.

---

### Task U8: Endings and results

**Files:** Create `src/client/UI/Ending.luau`, `src/client/UI/Results.luau`, `src/client/Controllers/EndingController.luau`.

- `Won` (12 s): control is disabled; the camera cuts to a fixed shot of the tower from the Yard (a `CameraShot_Win` marker — ask Environment to add it); the lens light sweeps on; the Keeper kneels and the coat falls empty; rain eases; fade to the title "The light holds."
- `Lost` (6 s): the local lamp dies; a slow push toward the Keeper's silhouette from a `CameraShot_Lose` marker in the Parlor; fade to "The sea keeps what it takes."
- Results (15 s): outcome, souls freed n / 10, difficulty, match time, XP gained with the rank bar animating, a "Returning to the dock in n" countdown.
- Skippable by nobody (short enough), but the Menu still opens.

- [ ] Follow the standard task cycle. T3: force each ending with DebugService.
- [ ] Commit: `feat: add endings and results`.

**End of M2 work for UI.**

---

### Task U9: Viewmodel and UI art pass

**Needs:** E10 (Storm Lamp model).

**Files:** Create `src/client/Controllers/ViewmodelController.luau`, `src/client/UI/Icons.luau`.

- First-person viewmodel: two arms and the Storm Lamp in the left hand, the held item in the right, rendered in the camera's space each frame. Sway from look speed (max 3°), bob with steps, lower when sprinting, lamp raised and shutter narrowed while the beam is on. The viewmodel never clips walls: it is drawn close (near 0.5 studs) and scaled so it stays inside the character's collision radius.
- The lamp's `SpotLight` for the local player is attached to the viewmodel lamp; for other players it is on the character's hand lamp.
- Item icons: rendered from the prefabs with a `ViewportFrame` once at start and cached.
- Art pass over every screen against the theme.

- [ ] Follow the standard task cycle. T3: screenshots while walking, sprinting, beaming, holding each item.
- [ ] Commit: `feat: add viewmodel and UI art`.

---

### Task U10: Settings

**Needs:** C14 (`SettingsSave`, `Profile`).

**Files:** Create `src/client/Controllers/SettingsController.luau`, `src/client/UI/Settings.luau`, `src/shared/Logic/SettingsRules.luau`. Test: `src/tests/unit/SettingsRules.spec.luau`.

```lua
SettingsRules.defaults(): {[string]: any}
SettingsRules.sanitize(input: {[string]: any}): {[string]: any}   -- unknown keys dropped, values clamped
```

| Key | Type | Default | Range |
|-----|------|---------|-------|
| `sensitivity` | number | 1.0 | 0.2–3.0 |
| `invertY` | boolean | false | |
| `headBob` | boolean | true | |
| `fov` | number | 70 | 60–90 |
| `photosensitive` | boolean | false | |
| `volMaster`, `volMusic`, `volEffects`, `volAmbience`, `volUi` | number | 0.8, 0.6, 0.8, 0.7, 0.7 | 0–1 |
| `holdToCrouch` | boolean | false | |
| `subtitles` | boolean | true | Text captions for key sounds: "[The Keeper screams]", "[Footsteps nearby]" |
| `matchesPlayed` | number | 0 | 0–9999. Used only for first-time hints (U13) |

Captions: SettingsController listens to the same `Cue` and `KeeperEvent`
remotes as audio and shows a caption line for these cues only:
`Keeper_Scream`, `Keeper_Alert`, `Keeper_Wake`, `WispRelease`,
`Brine_Emerge` (within 30 studs), `Door_Open_*` (within 18 studs, not the
local player's own). Caption text is in `Strings`.

**Tests:** `sanitize({sensitivity = 99}).sensitivity == 3.0`; `sanitize({nope = 1}).nope == nil`; `sanitize({headBob = "x"}).headBob == true` (falls back to the default); `sanitize({})` equals `defaults()`.

Settings apply at once, save through `SettingsSave` 1 s after the last change, and load from `Profile`. The `fov` setting replaces `GameConfig.Player.Fov` for the local camera; sprint adds 6.

- [ ] Follow the standard task cycle.
- [ ] Commit: `feat: add settings`.

---

### Task U11: Gamepad pass

- Every screen is fully usable with a gamepad: `GuiService.SelectedObject` is set on open; focus order is explicit (`NextSelectionDown` etc.); B closes; the selection frame uses the theme accent.
- Look sensitivity curve for sticks: dead zone 0.12, exponent 1.6, plus the `sensitivity` setting.
- Vibration: short pulse on grab wind-up, on being downed, on a successful stun.
- Prompts show gamepad glyphs when `device() == "Gamepad"` and switch live.

- [ ] T3 with the Studio controller emulator: complete the Q9 input matrix for gamepad.
- [ ] Commit: `feat: add full gamepad support`.

---

### Task U12: Touch controls

**Files:** Create `src/client/UI/TouchControls.luau`.

- Right side buttons: Interact (context, shows the action icon), Beam (hold), Lamp, Sprint (toggle), Crouch (toggle), Use, Drop. Layout avoids the Roblox jump button area (jump is hidden: `GuiService` / `TouchGui` jump button disabled).
- Buttons are 56 px, arranged in an arc within thumb reach; positions respect the safe area insets.
- Look: drag on the right half outside buttons. Sensitivity setting applies.
- The HUD scales so that no element overlaps a touch button at 844 × 390 and 667 × 375.

- [ ] T3 with the device emulator: complete the Q9 input matrix for touch.
- [ ] Commit: `feat: add touch controls`.

---

### Task U13: Perks and first-time hints

**Needs:** C14.

**Files:** Create `src/client/UI/Perks.luau`, `src/client/Controllers/HintController.luau`.

- Perks screen (from the lobby): rank and XP bar, tokens, six perk cards (name, effect from `Strings`, cost, Owned / Equipped state), up to two equipped. Buy and equip send `PerkBuy` and `PerkEquip`; the UI changes only when the `Profile` remote confirms. Errors show the reason (`NoTokens`, `Owned`, `TooMany`).
- Hints: on a player's first three matches (count kept in `settings.matchesPlayed`), show context hints once each: lamp toggle, beam on the first Keeper sighting, candle when first near a Wisp, pour at the Brazier with oil in hand, revive when a teammate is down and a dressing is held. Each hint is one line, 5 s, never during a chase.

- [ ] Follow the standard task cycle.
- [ ] Commit: `feat: add perks screen and hints`.
