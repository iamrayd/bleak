# Bleak Plan 03 — Physics and World Interaction

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Everything that touches the physics engine behaves the same way every time: collisions, zones, doors, first-person movement, dropped items, the carry proxy, and recovery from falls.

**Architecture:** Collision groups decide what touches what. Doors are driven by the server with tweens on anchored parts, never by hinges or forces. Anything that could fly, jam or tunnel is anchored, tweened or bounded.

**Tech Stack:** PhysicsService collision groups, TweenService, raycasts, PathfindingModifier.

**Spec:** `docs/spec/2026-11-13-bleak-game-design.md` sections 5.2, 8, 10.6, 14.

Agent: **Physics**. Every task uses the standard task cycle (master plan
section 7).

Physics principles for every workstream:

1. No unanchored part exists in the map. Only characters and dropped items simulate.
2. Stairs are ramps for collision: an invisible wedge or sloped part under visual steps. Slope ≤ 35°.
3. Doorways are at least 5 wide and 8 high. Corridors are at least 6 wide. This fits the Keeper (radius 2, height 7).
4. No part that characters walk on is thinner than 1 stud.
5. Nothing is moved by `BodyMover`, constraint or impulse.

---

### Task P1: Collision groups

**Files:**
- Modify: `src/server/Services/CollisionService.luau`
- Create: `src/shared/Logic/CollisionMatrix.luau`
- Test: `src/tests/unit/CollisionMatrix.spec.luau`

**Interfaces — Produces:** `CollisionService.setGroup(instance, group)` sets the group on the instance and every `BasePart` descendant, and on descendants added later.

```lua
CollisionMatrix.Groups: {string}     -- Default, Players, Npc, Wisp, Items, Doors, Proxy, Markers
CollisionMatrix.collides(a: string, b: string): boolean
```

Matrix (✓ collides):

| | Default | Players | Npc | Wisp | Items | Doors | Proxy | Markers |
|-|---------|---------|-----|------|-------|-------|-------|---------|
| Default | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | – | – |
| Players | | – | – | – | – | ✓ | – | – |
| Npc | | | – | – | – | ✓ | – | – |
| Wisp | | | | – | – | – | – | – |
| Items | | | | | ✓ | ✓ | – | – |
| Doors | | | | | | – | – | – |
| Proxy | | | | | | | – | – |
| Markers | | | | | | | | – |

Players do not collide with players, NPCs or items. This removes body
blocking, griefing in doorways and item kicking.

**Tests:** the matrix is symmetric; `collides("Players","Players")` false; `("Players","Doors")` true; `("Wisp","Doors")` false; `("Items","Default")` true; `("Markers", x)` false for every group; every pair of the 8 groups is defined.

**Behaviour:** `init` registers the groups and applies the matrix with `PhysicsService:CollisionGroupSetCollidable`. All parts under `Workspace.Map.Markers` get group `Markers`, `CanCollide = false`, `CanTouch = false`, `CanQuery = false`, `Transparency = 1`.

- [ ] Follow the standard task cycle.
- [ ] Commit: `feat: add collision groups`.

---

### Task P2: ZoneService

**Files:**
- Modify: `src/server/Services/ZoneService.luau`
- Create: `src/shared/Logic/ZoneMath.luau`
- Test: `src/tests/unit/ZoneMath.spec.luau`

```lua
-- type Box = { cframe: CFrame, size: Vector3, zoneId: ZoneId, floor: number }
ZoneMath.contains(box: Box, position: Vector3): boolean
ZoneMath.find(boxes: {Box}, position: Vector3): Box?      -- smallest-volume box that contains it
```

**Behaviour:** reads `Zone` tags at `init` into boxes. `zoneOf` and `floorOf` use `find`; `floorOf` returns 0 when no box matches. `isZoneOpen` is true when the zone is always-open, or when **any** `Door` with that `Zone` attribute is unlocked. Listens to `DoorService.doorChanged` and fires `zoneOpened` once per zone. `onMatchReset` clears the opened set.

**Tests:** a point inside a rotated box is contained; a point 0.1 outside is not; with nested boxes the smaller wins; no match returns nil.

- [ ] Follow the standard task cycle.
- [ ] Commit: `feat: add zone service`.

---

### Task P3: DoorService

**Needs:** C6 (InteractionService), P1.

**Files:**
- Modify: `src/server/Services/DoorService.luau`
- Create: `src/shared/Logic/DoorMath.luau`
- Test: `src/tests/unit/DoorMath.spec.luau`

```lua
DoorMath.swingSign(hingeCFrame: CFrame, openerPosition: Vector3): number     -- +1 or -1, away from the opener
DoorMath.isInside(insideSidePosition: Vector3, panelPosition: Vector3, actorPosition: Vector3): boolean
DoorMath.targetCFrame(hingeCFrame: CFrame, panelOffset: CFrame, angleDeg: number): CFrame
```

**Behaviour:**
- `Panel` is anchored, group `Doors`. Opening tweens the panel's CFrame around `Hinge` over 0.45 s (`Quad`, `Out`) to ±`OpenAngle`, sign from `swingSign`. During the tween the panel has `CanCollide = false`; when it ends, collision returns. A door never pushes anything.
- Closing: before starting, `Workspace:GetPartBoundsInBox` over the closed-panel volume grown by 1 stud, filter groups `Players` and `Npc`. If anything is there, wait and retry every 0.2 s for up to 3 s, then give up and stay open.
- Registers `OpenDoor`, `CloseDoor`, `Unlock` with InteractionService. `OpenDoor.canStart`: unlocked, or `isInside` (FS-12). `Unlock.onComplete`: re-check the key, `consume`, `unlock`, then open.
- Writes `Open` and `Locked`. Fires `doorChanged`. Emits noise `Door` (18), or `DoorSprint` (28) when the opener is sprinting.
- A door being tweened ignores new requests (returns `Notify("Busy")`).
- Each door has a `PathfindingModifier` on a thin invisible part filling the doorway: `PassThrough = true`, `Label = "Door"` when unlocked and `"LockedDoor"` when locked. `unlock` changes the label.
- `unlock(door)` unlocks **every** door that has the same non-empty `LockedBy` value (the Cellar and the Cannery have two doors each; one key opens the zone).
- `npcOpen(door, from, delay)`: after `delay`, if the door is still closed and unlocked, open it away from `from`. No effect on locked doors.
- `doorFor(part)`: the `Door` model that owns a part, or nil.
- `onMatchReset`: every door closed; `Locked = (LockedBy ~= "")`; labels restored.

**Tests:** `swingSign` is +1 for an opener on one side and −1 on the other, and never 0 (an opener exactly on the plane gets +1). `isInside` true when the actor and `InsideSide` are on the same side of the panel. `targetCFrame` at 0° equals hinge × offset; at 90° the panel centre is rotated 90° about the hinge's Y axis.

**Scenario checks (Review Focus 3):** two players press `OpenDoor` in the same frame: the door opens once and ends open. A player standing in the doorway stops it closing. A `Downed` player inside a locked zone can open its door. `Unlock` without the key is refused.

- [ ] Follow the standard task cycle.
- [ ] Commit: `feat: add door service`.

---

### Task P4: First-person movement and camera

**Needs:** U1 (InputController) for bindings. Until U1 lands, read `UserInputService` directly behind a local function that U1 later replaces.

**Files:**
- Create: `src/client/Controllers/MovementController.luau`, `CameraController.luau`
- Create: `src/shared/Logic/CameraMath.luau`
- Test: `src/tests/unit/CameraMath.spec.luau`

```lua
CameraMath.bob(t: number, speed: number, amplitude: number): Vector3     -- offset; zero when speed < 0.5
CameraMath.fovFor(sprinting: boolean, base: number, sprint: number): number
CameraMath.smooth(current: number, target: number, dt: number, halfLife: number): number
```

**Behaviour:**
- `Player.CameraMode = LockFirstPerson`. `CameraMaxZoomDistance = 0.5`. Mouse icon hidden while playing; shown when a menu is open.
- Local character parts are hidden except the viewmodel (U9). Other players' characters are visible.
- Sends `SetMove(sprint, crouch)` only when a value changes. Predicts speed locally for feel: sets nothing on the Humanoid; the server's `WalkSpeed` is the truth. The local FOV follows the `Sprinting` attribute, not the key.
- FOV tween 0.25 s. Head bob from `CameraMath.bob` with amplitudes 0.08 and 0.14, applied through `Humanoid.CameraOffset`. Off when the setting says so or status is not `Alive`.
- Crouch lowers `CameraOffset.Y` by 1.5 with `smooth`, half-life 0.08 s.
- While `Carried`: camera type `Scriptable`, following the proxy's head from `CarryView`, with a 0.6 s fade to black at the start and end.
- While `Downed`: camera 1.5 lower, roll 8°.
- While `Spectating`: handled by SpectateController (U7).
- Mouse sensitivity and invert-Y come from settings.

**Tests:** `bob(t, 0, 0.08)` is zero; `bob` magnitude never exceeds amplitude; `fovFor(true, 70, 76) == 76`; `smooth(0, 10, 0.08, 0.08) == 5` (within 1e-3); `smooth` with dt 0 returns current.

**Smoothness checks (T3, Game Tester in Q10):** no camera stutter when turning at 60 FPS; no visible snap when sprint starts or stops; crouch does not clip the camera through floors or low ceilings.

- [ ] Follow the standard task cycle.
- [ ] Commit: `feat: add first-person movement and camera`.

**End of M1 work for Physics.**

---

### Task P5: Item physics

**Files:**
- Create: `src/server/Services/ItemPhysics.luau`
- Create: `src/shared/Logic/DropMath.luau`
- Test: `src/tests/unit/DropMath.spec.luau`

```lua
ItemPhysics.place(item: Model, nearPosition: Vector3)        -- called by ItemService.spawnAt for drops
DropMath.dropPoint(root: Vector3, look: Vector3): Vector3    -- 2 studs ahead, 1 up
```

**Behaviour:** an item never simulates freely. `place` raycasts down 20 studs from `dropPoint` against group `Default`. On a hit, the item is anchored resting on the surface with a random yaw, tweened there in 0.15 s. If the forward point is inside geometry (raycast from the root to the drop point is blocked), drop at the root's feet instead. On no ground hit, the item goes to its spawn point (FS-2). Items inside a door's swing volume are moved 2 studs toward the dropper. An item is at least 1.5 studs from any other item: on overlap, step outward in a spiral of up to 8 positions.

**Tests:** `dropPoint((0,3,0), (0,0,-1)) == (0,4,-2)`; a look vector with a Y part is flattened first; a zero look vector uses (0,0,-1).

**Scenario checks:** drop against a wall leaves the item on the player's side; drop over the sea returns the item to its spawn point; 8 items dropped on the same spot do not overlap.

- [ ] Follow the standard task cycle.
- [ ] Commit: `feat: add deterministic item placement`.

---

### Task P6: Carry proxy

**Needs:** M4 (Keeper model with a `CarryAttachment` on the upper torso).

**Files:**
- Modify: `src/server/AI/CarryProxy.luau`

**Behaviour of `begin(victim, keeper)`:**
1. Clone `ServerStorage.Prefabs.CarryProxy` (a limp R15 dummy, group `Proxy`, all parts massless, `CanCollide = false`).
2. Copy the victim's appearance with `Humanoid:ApplyDescription(Players:GetHumanoidDescriptionFromUserId(...))` inside `pcall`; on failure keep the grey default.
3. Weld the proxy's root to `CarryAttachment`. Parent to `Workspace.Runtime.Proxies`.
4. Hide the real character: anchor the root, set every part `Transparency = 1` and `CanCollide = false`, move it to a holding CFrame 2,000 studs **above** the map centre. (Never below: `Workspace.FallenPartsDestroyHeight` destroys parts under it.) RecoveryService ignores `Carried` players.
5. Fire `CarryView(proxy)` to the victim. Return the proxy.

**Behaviour of `finish(victim, dropCFrame)`:** destroy the proxy; move the real character to `dropCFrame`; restore transparency and collision from values saved in `begin`; unanchor; fire `CarryView(nil)`. If the victim has left, only destroy the proxy.

`finish` is safe to call twice. Every `begin` has exactly one matching
`finish`: KeeperService calls it on drop, on stun, on path failure, on match
reset, and when `statusChanged` reports the victim left `Carried`.

**Scenario checks:** after a carry, the victim's character is visible, collides, is not anchored and stands at the drop point. A victim who disconnects mid-carry leaves no proxy (FS-5). A match reset mid-carry leaves no proxy.

- [ ] Follow the standard task cycle.
- [ ] Commit: `feat: add carry proxy`.

---

### Task P7: Recovery and movement checks

**Files:**
- Modify: `src/server/Services/RecoveryService.luau`
- Create: `src/shared/Logic/SpeedCheck.luau`
- Test: `src/tests/unit/SpeedCheck.spec.luau`

```lua
-- type SpeedState = { strikes: number }
SpeedCheck.step(s: SpeedState, horizontalDistance: number, dt: number, allowedSpeed: number)
  : (SpeedState, boolean)                        -- new state, shouldCorrect
```

Rule: a sample breaks the limit when distance / dt > allowedSpeed × 1.35 + 2.
Three breaking samples in a row give `shouldCorrect`. A clean sample resets
strikes.

**Behaviour:**
- 5 Hz loop per player with a character and status `Alive` or `Downed`:
  - Root Y < `KillY`, or root outside the map bounds box (the union of `Zone` boxes grown by 60): respawn at `nearestSafeNode(lastGoodPosition)`, keeping status (FS-3).
  - `SpeedCheck.step` with the allowed speed from `PlayerStatusRules.speedFor`. On `shouldCorrect`, set the root CFrame back to the last good position and log it.
  - Store the position as last good when the sample was clean and the root is on the ground.
- Skips players who were moved by the server in the last 1 s (spawn, carry finish).
- 1 Hz loop over items: any item below `KillY` or outside bounds returns to its spawn point (FS-2).
- `nearestSafeNode` picks the nearest `SafeNode` in an open zone.

**Tests:** three samples at 30 studs/s with allowed 17 → `shouldCorrect` on the third. Samples 30, 10, 30, 30 → no correction. Allowed 17: the limit is 24.95, so 24 studs/s never corrects.

**Scenario checks (Review Focus 2):** teleport a character to Y = −100: it reappears at a safe node within 1 s, same status, same held item. `Humanoid.Health = 0` on a `Downed` player: the new character is `Downed`. Killing a `Carried` player's hidden character does not break the carry.

- [ ] Follow the standard task cycle.
- [ ] Commit: `feat: add recovery and speed checks`.
