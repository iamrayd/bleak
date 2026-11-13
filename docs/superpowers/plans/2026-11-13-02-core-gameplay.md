# Bleak Plan 02 — Core Gameplay

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** The rules of a match: phases, player status, stamina, lamp, items, interactions, the Brazier, noise, disconnects, difficulty, solo play and progression.

**Architecture:** Each rule is a pure function in `src/shared/Logic` with a T1 spec. Each service holds state, calls the rule, and writes Attributes. Services talk through the interfaces in master plan section 6.

**Tech Stack:** Luau strict, TestKit, DataStoreService.

**Spec:** `docs/spec/2026-11-13-bleak-game-design.md` sections 4–9, 12–15.

Agent: **Core**. Every task uses the standard task cycle (master plan
section 7). "State" tables below are plain Luau tables; rule functions never
mutate their input and always return a new table.

---

### Task C1: Stage table, stamina and lamp charge rules

**Files:**
- Create: `src/shared/Logic/StageTable.luau`, `Stamina.luau`, `LampCharge.luau`
- Test: `src/tests/unit/StageTable.spec.luau`, `Stamina.spec.luau`, `LampCharge.spec.luau`

**Interfaces — Produces:**

```lua
StageTable.get(stage: number, difficulty: Difficulty): StageParams
-- stage is clamped to 0..9. Applies DifficultyConfig: speeds × keeperSpeed,
-- stunTime × stunTime factor. Fills enragedSpeed = chaseSpeed + 1.5 (after the multiplier),
-- carrySpeed = roamSpeed + 2, emberImmunity = 3.0 when stage >= 7 else 0.

-- type StaminaState = { value: number, exhausted: boolean, sinceSprint: number }
Stamina.new(): StaminaState                         -- 100, false, math.huge
Stamina.step(s: StaminaState, dt: number, wantsSprint: boolean, moving: boolean)
  : (StaminaState, boolean)                         -- new state, isSprinting

-- type LampState = { charge: number, locked: boolean, sinceBeam: number, max: number }
LampCharge.new(max: number): LampState              -- charge = max
LampCharge.step(s: LampState, dt: number, beamRequested: boolean, beamAllowed: boolean)
  : (LampState, boolean)                            -- new state, beamActive
LampCharge.refill(s: LampState, amount: number): LampState
```

**Tests:**
- StageTable: `get(0,"Normal").chaseSpeed == 13.5`; `.enragedSpeed == 15.0`; `.carrySpeed == 9.0`. `get(9,"Hard").chaseSpeed` ≈ 19.44; `.stunTime` ≈ 1.5. `get(7,"Normal").emberImmunity == 3.0`; `get(6,...)` is 0. `get(12,...)` equals `get(9,...)`; `get(-1,...)` equals `get(0,...)`. `get(4,"Nightmare").stunTime == 0`.
- Stamina: sprinting 2.0 s from 100 gives 60. Sprinting 5.0 s gives 0 and `exhausted`. While exhausted, `wantsSprint` returns `isSprinting == false`. After sprint stops at value 0, no regen for 1.5 s; 2.5 s after stopping the value is 12. From 0, `exhausted` clears when value ≥ 25 (about 3.59 s after stopping). `wantsSprint` with `moving == false` does not drain and returns false. Value never leaves 0..100 with dt = 10.
- LampCharge: beam 2.0 s from 100 gives 72. Beam until 0 sets `locked`; a locked lamp returns `beamActive == false`. No regen until 3.0 s after the beam stops; then 2.5 per second. `locked` clears at ≥ 15. `beamAllowed == false` never drains. `refill(s, 60)` from 70 with `max = 100` gives 100; from 70 with `max = 125` gives 125; from 20 with `max = 100` gives 80. `refill` on a locked lamp with result ≥ 15 clears `locked`.

- [ ] Follow the standard task cycle. Step the functions in tests with dt = 0.1 in a loop.
- [ ] Commit: `feat: add stage, stamina and lamp charge rules`.

---

### Task C2: PlayerStateService

**Needs:** P1 (collision group name only; call `CollisionService.setGroup`).

**Files:**
- Modify: `src/server/Services/PlayerStateService.luau` (replace stub)
- Test: `src/tests/unit/PlayerStatusRules.spec.luau`; scenario `src/tests/scenarios/PlayerState.scenario.luau` (Game Tester writes it in Q4 from the list below)
- Create: `src/shared/Logic/PlayerStatusRules.luau`

**Interfaces — Produces:** master plan section 6, `PlayerStateService`. Plus:

```lua
PlayerStatusRules.canTransition(from: PlayerStatus, to: PlayerStatus): boolean
-- Allowed: Lobby->Alive, Lobby->Spectating, Spectating->Lobby, Alive->Carried, Alive->Downed,
-- Carried->Downed, Downed->Alive, and any->Lobby. Everything else false. Same->same false.
PlayerStatusRules.speedFor(status: PlayerStatus, sprinting: boolean, crouched: boolean,
  fleetfoot: boolean, chased: boolean): number
```

**Behaviour:**
- `setStatus` applies `canTransition`; an illegal transition logs a warning and does nothing. It writes the `Status` attribute, fires `statusChanged`, and sets `Humanoid.WalkSpeed` from `speedFor`.
- `spawnCharacter`: `player:LoadCharacter()`, then pivot to `at`, set `JumpHeight = 0`, `UseJumpPower = false`, `MaxHealth = 100` (connect `HealthChanged` to set health back to 100), put every part in group `Players`, set `Humanoid.DisplayDistanceType = None`.
- Handles `SetMove(sprint, crouch)`: validated with `Validate.boolean`. Ignored unless status is `Alive`. Stores wishes; a 20 Hz loop calls `Stamina.step` for each `Alive` player with `moving` = horizontal root velocity > 0.5, then writes `Stamina` (rounded), `Exhausted`, `Sprinting`, `Crouched` only when changed.
- Crouch sets `Humanoid.HipHeight` lower by 1.0 and speed to crouch speed. Sprint and crouch together: crouch wins.
- On `CharacterRemoving` or `Humanoid.Died` during a match: calls `RecoveryService.nearestSafeNode` and respawns the character there with the same status (FS-3).
- `onMatchReset`: every player still in the server gets status `Lobby` and a character at a `LobbySpawn`.

**Tests** (unit): `canTransition("Alive","Carried")` true; `("Downed","Carried")` false; `("Carried","Alive")` false; `("Alive","Alive")` false; any → `"Lobby"` true. `speedFor("Alive", true, false, false, false) == 17`; with `fleetfoot` and `chased` ≈ 19.55; `("Alive", true, true, ...) == 5`; `("Downed", true, false, ...) == 3`; `("Carried", ...) == 0`.

**Scenario checks** (T2): sprint for 5 s drains the attribute to 0 and speed falls to 10. `SetMove` from a `Downed` player changes nothing. 100 `SetMove` calls in one frame cause no error and at most 20 are processed. `SetMove("x", 1)` is dropped.

- [ ] Follow the standard task cycle.
- [ ] Commit: `feat: add player state service`.

---

### Task C3: Match rules, MatchService and LobbyService

**Files:**
- Create: `src/shared/Logic/MatchRules.luau`
- Modify: `src/server/Services/MatchService.luau`, `LobbyService.luau`
- Test: `src/tests/unit/MatchRules.spec.luau`

**Interfaces — Produces:** master plan section 6, plus:

```lua
MatchRules.evaluate(statuses: {PlayerStatus}, lastBreathPending: boolean, released: number)
  : "Continue" | "Won" | "Lost"
MatchRules.canStart(readyFlags: {boolean}, phase: MatchPhase): boolean
MatchRules.nextHost(joinTimes: {[number]: number}): number?    -- userId with the lowest time
```

**Behaviour:**
- Phases and timings exactly as spec 4.2, using `GameConfig.Match`. Each phase change writes `Phase` and `PhaseEndsAt` and fires `phaseChanged(new, old)`.
- `Lobby → Intro`: `LobbyStart` from the host and `canStart` true. The RNG seed is `os.time()` and is logged.
- `Intro`: lobby players become `Alive` and are spawned at `PlayerSpawn` points (one each, shuffled). `Solo` is set when exactly one. `ItemService.spawnAll()` is called.
- `LobbyGate`: the tagged part has `CanCollide = true` in `Lobby` and `Results`, and `false` in every other phase, so lobby players stay on the pier.
- `Dormant → Active`: `wake(reason)`; idempotent; also fired by the 240 s timeout.
- Stage: listens to `BrazierService.wispReleased`; writes `Stage` and `Released`; fires `stageChanged`.
- Every `statusChanged`, release and `PlayerRemoving` re-runs `evaluate`. `Won` is checked before `Lost`.
- `Results`: sends the `Results` remote, calls `DataService.addXp`, waits 15 s, then calls `onMatchReset()` on every service in init order, destroys all children of `Workspace.Runtime`, and enters `Lobby`.
- LobbyService: tracks join time, `IsHost`, `Ready`. Handles `LobbyReady`, `LobbyDifficulty` (host only, `Validate.oneOf`), `LobbyStart` (host only). Clears `Ready` on reset. When the host leaves, `nextHost` picks the new one (FS-15). A player who joins outside `Lobby` gets `Spectating` (FS-16).

**Tests:**
- `evaluate({"Alive","Downed"}, false, 3) == "Continue"`; `({"Downed","Carried"}, false, 3) == "Lost"`; `({"Downed"}, true, 3) == "Continue"`; `({"Downed"}, false, 10) == "Won"`; `({}, false, 0) == "Lost"`; `({"Spectating"}, false, 2) == "Lost"`.
- `canStart({true, true}, "Lobby")` true; `({true, false}, "Lobby")` false; `({}, "Lobby")` false; `({true}, "Active")` false.
- `nextHost({[5]=10, [7]=3}) == 7`; `nextHost({}) == nil`.

- [ ] Follow the standard task cycle.
- [ ] Commit: `feat: add match flow and lobby`.

---

### Task C4: LampService

**Files:**
- Modify: `src/server/Services/LampService.luau`
- Create: `src/shared/Logic/BeamMath.luau`
- Test: `src/tests/unit/BeamMath.spec.luau`

**Interfaces — Produces:** master plan section 6, plus:

```lua
BeamMath.inCone(origin: Vector3, direction: Vector3, target: Vector3,
  halfAngleDeg: number, range: number): boolean
```

**Behaviour:**
- Handles `SetLamp`, `SetBeam`, `AimUpdate` (`Validate.unitVector`). The latest aim per player is stored; an aim older than 0.5 s makes `getBeam` return nil.
- `beamAllowed` = status `Alive` and lamp on and `DifficultyConfig[difficulty].beam`.
- A 10 Hz loop: `LampCharge.step` per player; writes `LampOn`, `BeamOn`, `LampCharge` (rounded), `LampLocked` on change. For each player with an active beam and each registered target: if `inCone` and a raycast from the beam origin to the target hits nothing in groups `Default` or `Doors` first, call `onDwell(player, 0.1)`.
- Beam origin = `PlayerStateService.getHeadPosition`.
- Lamp max is 125 when `DataService.hasPerk(player, "DeepReservoir")`.
- Status leaving `Alive` forces lamp and beam off. `onMatchReset` restores full charge.

**Tests:** `inCone` with origin 0, direction +Z: target (0,0,10) true; (0,0,29) false at range 28; a target 9° off axis at distance 10 true; 10° false; target at the origin false.

**Scenario checks:** holding beam 8 s locks the lamp and `BeamOn` becomes false; a registered dummy target behind a closed door receives no dwell; on Nightmare `SetBeam(true)` never sets `BeamOn`.

- [ ] Follow the standard task cycle.
- [ ] Commit: `feat: add lamp service`.

---

### Task C5: Spawn planning, items and inventory

**Needs:** P1, P5 for final drop physics (until P5 lands, a dropped item is anchored at the feet).

**Files:**
- Create: `src/shared/Logic/KeyPlacement.luau`, `SpawnPlanner.luau`, `InventoryRules.luau`
- Modify: `src/server/Services/ItemService.luau`, `InventoryService.luau`
- Test: `src/tests/unit/KeyPlacement.spec.luau`, `SpawnPlanner.spec.luau`, `InventoryRules.spec.luau`

**Interfaces — Produces:** master plan section 6, plus:

```lua
-- type Point = { id: string, kinds: {ItemKind}, zone: ZoneId }
-- type ZoneInfo = { alwaysOpen: boolean, lockedBy: ItemKind? }
KeyPlacement.isSolvable(keyZones: {[ItemKind]: ZoneId}, zones: {[ZoneId]: ZoneInfo}): boolean
SpawnPlanner.plan(rng: Rng, points: {Point}, zones: {[ZoneId]: ZoneInfo})
  : {[string]: ItemKind}                     -- pointId -> kind
SpawnPlanner.pickRespawnPoint(rng: Rng, kind: ItemKind, points: {Point},
  occupied: {[string]: boolean}): string?
InventoryRules.afterPickUp(held: ItemKind?, picked: ItemKind): (ItemKind, ItemKind?)  -- newHeld, dropped
```

**Behaviour:**
- `plan`: places keys first, retrying up to 50 draws until `isSolvable`; if none is found it uses the first candidate point of each key in an always-open zone and logs a warning. Then places the other kinds with the counts in `ItemConfig`. One item per point. At least 6 `OilCanister` in always-open zones.
- ItemService: reads `ItemSpawn` tags once at `start`. `spawnAt` clones `ServerStorage.Prefabs.Items[kind]` into `Workspace.Runtime.Items`, sets `Kind`, group `Items`, registers a `PickUp` interaction. Tracks each item's spawn point for FS-2. `count` = world + held.
- InventoryService: `pickUp` is atomic: it checks the item still exists and is not held, removes the model, sets `HeldItem`. The second of two same-tick calls returns false and that player gets `Notify("Busy")`. `drop` re-creates the model at the root position with the item's state (candle lit time is kept). `consume` clears the slot and returns the kind.
- `DropItem` remote → `drop`. Status leaving `Alive` → `drop`.
- `UseItem`: `VotiveCandle` unlit → lit (`CandleLit`, `CandleEndsAt = now + burn`). `LampFlask` → `LampService.refill`, consume. Other kinds: nothing.

**Tests:**
- `isSolvable`: Cannery key in HouseGround, Tower key in Cannery, others in Yard → true. Cannery key in Cannery → false. Cannery key in Tower and Tower key in Cannery → false.
- `plan` for seeds 1..200 with a 26+16+10+12+22-point fixture: always solvable; counts equal `ItemConfig` initial counts; no point used twice; every kind sits on a point that allows it; ≥ 6 oil in always-open zones.
- `pickRespawnPoint` never returns an occupied point; returns nil when all are occupied.
- `afterPickUp(nil, "LampFlask")` → `("LampFlask", nil)`; `("OilCanister", "KeyTower")` → `("KeyTower", "OilCanister")`.

**Scenario checks (Review Focus 3):** two players call `Interact` on the same item in the same frame: exactly one holds it, `ItemService.count` is unchanged. Swap leaves the old item on the floor. A lit candle dropped and picked up keeps its `CandleEndsAt`.

- [ ] Follow the standard task cycle.
- [ ] Commit: `feat: add item spawning and inventory`.

---

### Task C6: Prompt rules and InteractionService

**Files:**
- Create: `src/shared/Logic/PromptRules.luau`
- Modify: `src/server/Services/InteractionService.luau`
- Test: `src/tests/unit/PromptRules.spec.luau`

**Interfaces — Produces:**

```lua
-- type TargetInfo = { tags: {string}, attrs: {[string]: any}, isPlayer: boolean, playerStatus: PlayerStatus? }
-- type ActorInfo = { status: PlayerStatus, held: ItemKind?, candleLit: boolean, perks: {[PerkId]: boolean} }
-- type World = { brazierLit: boolean, brazierCharges: number }
-- type Prompt = { action: string, hold: number, labelCode: string }
PromptRules.resolve(target: TargetInfo, actor: ActorInfo, world: World): Prompt?
```

Resolution table (first match wins; nil when the actor is not `Alive`):

| Target | Condition | action | hold |
|--------|-----------|--------|------|
| Item model | — | `PickUp` | 0 |
| Door, `Locked` and actor holds its `LockedBy` key | — | `Unlock` | 1.5 |
| Door, `Locked` and actor is on the inside | — | `OpenDoor` | 0 |
| Door, `Locked` otherwise | — | nil; label `Locked` is shown as a hint only |
| Door, unlocked | `Open` false / true | `OpenDoor` / `CloseDoor` | 0 |
| Cage | `Opened` false | `OpenCage` | 2.0 |
| Brazier `Bowl` | holds `OilCanister` and (not lit or charges < 3) | `Pour` | 3.0 |
| Brazier `Bowl` | lit and charges ≥ 1 | `Release` | 2.5 |
| Sconce | holds lit `VotiveCandle` and sconce empty | `PlaceCandle` | 1.0 |
| Player character | target `Downed`, actor holds `FieldDressing` | `Revive` | 4.0 (2.0 with `FieldMedic`) |

Every hold above 0 is multiplied by 0.70 with `SteadyHands`, except `Revive`
when `FieldMedic` already applies (use the lower of the two results).

**Behaviour of InteractionService:**
- `Interact(target, "Begin")`: validate `Validate.instance`; find the registered ancestor; build `TargetInfo`/`ActorInfo`; `resolve`; check range ≤ 7 from the head and a clear raycast (ignoring the target and the actor); call the handler's `canStart`. On failure send `Notify(code)`.
- Hold 0: call `onComplete` at once.
- Hold > 0: lock the target (one holder), send `HoldProgress`, tick `onTick` each second if present, complete after the duration if still valid. Cancel on `"Cancel"`, move > 2 studs, status change, target destroyed, or the actor leaving (FS-6). On cancel send `HoldProgress(nil, ...)` and unlock.
- A player has at most one hold.

**Tests:** one test per table row, plus: a `Downed` actor always gets nil; `Pour` at 3 charges falls through to `Release`; `Pour` unlit with a canister is offered; a locked door from the inside offers `OpenDoor`; `SteadyHands` gives `Pour` hold 2.1; `FieldMedic` + `SteadyHands` gives `Revive` 2.0.

**Scenario checks:** a second player beginning the same hold gets `Notify("Busy")`; walking 3 studs cancels; a forged `Interact` on a target 50 studs away does nothing.

- [ ] Follow the standard task cycle.
- [ ] Commit: `feat: add prompt rules and interaction service`.

---

### Task C7: Brazier

**Needs:** C5, C6. Uses `WispService.nearestWisp` and `WispService.release` (stub until M8).

**Files:**
- Create: `src/shared/Logic/BrazierRules.luau`
- Modify: `src/server/Services/BrazierService.luau`
- Test: `src/tests/unit/BrazierRules.spec.luau`

**Interfaces — Produces:**

```lua
BrazierRules.new(): BrazierState                               -- false, 0, 0
BrazierRules.pour(s: BrazierState): (BrazierState, boolean, string?)     -- ok, reason "Full"
BrazierRules.release(s: BrazierState, wispInRange: boolean)
  : (BrazierState, boolean, string?)        -- reasons "NotLit", "NoFuel", "NoWisp", "Done"
```

**Behaviour:**
- Registers `Pour` and `Release` on `Bowl`, `PlaceCandle` on each sconce.
- `Pour.onComplete`: re-check the actor still holds an `OilCanister`; `consume`; apply `pour`; on first light fire `ignited` and call `MatchService.wake("BrazierLit")`. `onTick`: `NoiseService.emit(bowl, NoiseConfig.Pour, "Pour", player)`.
- `Release.onComplete`: `nearestWisp(bowl, 5)`; apply `release`; on ok call `WispService.release(id)`, fire `wispReleased(released)`, emit a whole-map noise (radius 10,000).
- `PlaceCandle.onComplete`: consume the candle; set the sconce's `CandleEndsAt` to the candle's end time; when it passes, set 0. `litSconcePositions` returns sconces whose time is in the future.
- Writes `BrazierLit`, `BrazierCharges`. `onMatchReset` → `new()` and empty sconces.

**Tests:** pour on new → lit, charges 0, ok. Three more pours → charges 3. A fourth → not ok, `"Full"`, state unchanged. Release unlit → `"NotLit"`. Lit with 0 charges → `"NoFuel"`. Charges 1 and no wisp → `"NoWisp"`, charges still 1. Charges 1 with wisp → charges 0, released 1. Released 10 → `"Done"`. Eleven pours interleaved with ten releases reach released 10.

**Scenario checks (Review Focus 3):** two players complete `Release` in the same tick with 1 charge and 2 wisps: released rises by exactly 1. Two players pour with charges at 2: one succeeds, the other keeps the canister and gets `Notify("Full")`.

- [ ] Follow the standard task cycle.
- [ ] Commit: `feat: add brazier rules and service`.

---

### Task C8: Downed and revive

**Files:**
- Modify: `src/server/Services/PlayerStateService.luau` (register `Revive` on each character)
- Test: scenario list below

**Behaviour:**
- On `Alive → Downed`: drop the held item, lamp off, play the downed pose (animation hook; placeholder until M12), speed 3.
- `Revive.canStart`: target is `Downed`, actor holds `FieldDressing`. `onTick`: emit `NoiseConfig.Revive`. `onComplete`: re-check both conditions, `consume`, target → `Alive`, stamina 50.
- If the target leaves, is carried or revives otherwise during the hold, the hold cancels and the dressing is kept.

**Scenario checks:** revive completes in 4.0 s ± 0.2; the dressing is gone; stamina is 50. The reviver being grabbed mid-hold cancels and keeps the dressing. The target disconnecting mid-hold cancels with no error (Review Focus 1).

- [ ] Follow the standard task cycle.
- [ ] Commit: `feat: add downed state and revive`.

---

### Task C9: Noise and detection math

**Files:**
- Create: `src/shared/Logic/DetectionMath.luau`
- Modify: `src/server/Services/NoiseService.luau`
- Test: `src/tests/unit/DetectionMath.spec.luau`

**Interfaces — Produces:**

```lua
DetectionMath.visionRange(beamOn: boolean, lampOn: boolean, moving: boolean, crouched: boolean): number
DetectionMath.inVisionCone(eye: Vector3, look: Vector3, target: Vector3, range: number): boolean
DetectionMath.heard(distance: number, radius: number, sameFloor: boolean): boolean
```

**Behaviour of NoiseService:** `emit` multiplies the radius by 0.75 when the source has `QuietSoles`, drops radius 0, and fires `noise`. A 20 Hz loop emits footsteps: sprinting players every 0.35 s at `SprintStep`, walking every 0.5 s at `WalkStep`, crouched none. NoiseService sends no remotes: footstep sounds are made on the client (plan 06, task A4).

**Tests:** `visionRange(true, true, true, false) == 60`; `(false, true, false, true) == 45`; `(false, false, true, false) == 22`; `(false, false, true, true) == 10`; `(false, false, false, false) == 10`. `inVisionCone` 54° off axis true, 56° false, beyond range false. `heard(30, 30, true)` true; `(18.1, 30, false)` false; `(18, 30, false)` true.

- [ ] Follow the standard task cycle.
- [ ] Commit: `feat: add noise service and detection math`.

**End of M1 work for Core.**

---

### Task C10: Disconnects and late joins

**Files:**
- Modify: `MatchService`, `InventoryService`, `InteractionService`, `LobbyService`
- Test: scenarios in Q6

**Behaviour:** on `PlayerRemoving`, in this order: cancel the player's hold (FS-6); `InventoryService.drop(player, "Left")` at the last known root position (FS-4); `LampService` and `RateLimiter:forget`; fire `statusChanged(player, "Lobby", old)` so the Keeper and CarryProxy clean up (FS-5); re-run `evaluate` (FS-7). When `#Players:GetPlayers() == 0` during a match, go straight to reset (FS-8). `PlayerStateService` caches each player's last root position at 5 Hz so a drop position exists after the character is gone.

- [ ] Follow the standard task cycle. The Game Tester writes the Q6 scenarios first; this task makes them pass.
- [ ] Commit: `fix: handle disconnects and late joins`.

---

### Task C11: Difficulty

**Files:**
- Modify: `MatchService` (`getStageParams` uses `StageTable.get(stage, difficulty)`), `ItemService` (respawn interval × factor; dressing never respawns on Nightmare)
- Test: add to `StageTable.spec.luau` and a new `src/tests/unit/Respawn.spec.luau`
- Create: `src/shared/Logic/RespawnRules.luau`

```lua
RespawnRules.due(kind: ItemKind, existing: number, sinceLast: number, difficulty: Difficulty): boolean
```

**Tests:** `due("VotiveCandle", 3, 75, "Normal")` true; `(…, 4, 999, "Normal")` false; `(…, 3, 100, "Hard")` false; `(…, 3, 112.5, "Hard")` true; `due("FieldDressing", 0, 9999, "Nightmare")` false; `due("OilCanister", 0, 9999, "Normal")` false.

- [ ] Follow the standard task cycle.
- [ ] Commit: `feat: apply difficulty modifiers`.

---

### Task C12: Solo Last Breath

**Files:** Modify `MatchService`, `PlayerStateService`. Test: add to `MatchRules.spec.luau`; scenario in Q5.

**Behaviour:** at `Intro` with one player, `LastBreath` = `DifficultyConfig[d].lastBreath`. On `Downed` with charges > 0: mark pending, wait 15 s, then if still `Downed` → `Alive`, stamina 50, charge − 1. `evaluate` receives `lastBreathPending`. Being carried does not start the wait; it starts when the status becomes `Downed`.

**Tests:** already in C3 (`({"Downed"}, true, 3) == "Continue"`). Add: pending with released 10 is `"Won"`.

- [ ] Follow the standard task cycle.
- [ ] Commit: `feat: add solo last breath`.

---

### Task C13: Item respawn and candle lifecycle

**Files:** Modify `ItemService`, `InventoryService`. Test: `Respawn.spec.luau` (C11), scenario in Q5.

**Behaviour:** a 1 Hz loop: for each respawning kind, when `RespawnRules.due`, spawn one at `pickRespawnPoint`. A lit candle whose end time passes is destroyed wherever it is (hand, floor, sconce) and `CandleLit` clears. `TrimmedWick` sets burn time 160 for candles lit by that player.

- [ ] Follow the standard task cycle.
- [ ] Commit: `feat: add item respawn and candle burn`.

**End of M2 work for Core.**

---

### Task C14: Progression and data

**Files:**
- Create: `src/shared/Logic/XpRules.luau`, `PerkRules.luau`
- Modify: `src/server/Services/DataService.luau`
- Test: `src/tests/unit/XpRules.spec.luau`, `PerkRules.spec.luau`

```lua
XpRules.matchXp(won: boolean, released: number, difficulty: Difficulty): number
XpRules.rankForXp(xp: number): number
XpRules.tokens(rank: number, ownedPerks: {PerkId}): number       -- rank - 2 × #owned
PerkRules.canBuy(tokens: number, owned: {PerkId}, perk: string): (boolean, string?)
PerkRules.canEquip(owned: {PerkId}, wanted: {string}): (boolean, string?)
```

**Behaviour of DataService:** store `BleakProfile_v1`, key `u_<userId>`. Load with `GetAsync` on join, 3 tries with 2 s, 4 s, 8 s waits. If loading fails, the player plays with a default profile flagged `unsaved` and nothing is written for them (no data loss by overwrite). Save with `UpdateAsync` on leave, at match end, and in `game:BindToClose` (wait for all saves, at most 25 s). The profile is validated on load: unknown perks are dropped, numbers are clamped ≥ 0. Sends `Profile` after load and after each change. Handles `PerkBuy`, `PerkEquip`, `SettingsSave` (settings: at most 20 keys, values boolean or number).

**Tests:** `matchXp(true, 10, "Normal") == 200`; `(true, 10, "Hard") == 250`; `(true, 10, "Nightmare") == 350`; `(false, 4, "Hard") == 40`. `rankForXp(0) == 0`, `(149) == 0`, `(150) == 1`, `(449) == 1`, `(450) == 2`, `(900) == 3`. `tokens(3, {"Fleetfoot"}) == 1`. `canBuy(1, {}, "Fleetfoot")` false `"NoTokens"`; `canBuy(2, {"Fleetfoot"}, "Fleetfoot")` false `"Owned"`; `canBuy(2, {}, "Nope")` false `"Unknown"`. `canEquip({"Fleetfoot"}, {"Fleetfoot","QuietSoles"})` false `"NotOwned"`; three perks false `"TooMany"`; duplicates false.

- [ ] Follow the standard task cycle. In Studio, enable "Studio Access to API Services" (ask the user) or use a mock store when `RunService:IsStudio()` and the setting is off.
- [ ] Commit: `feat: add progression and saved data`.
