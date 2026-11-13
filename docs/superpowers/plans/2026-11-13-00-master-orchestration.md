# Bleak Master Orchestration Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build Bleak, an original 1–4 player co-op first-person horror game for Roblox, with realistic graphics, smooth play and no known bugs at release.

**Architecture:** One Roblox place where one server runs one match. Rules live in pure Luau modules that are unit-tested without the engine; thin server services apply them to instances; clients render and send requests. The map is generated at edit time from data files so that it is reproducible from git.

**Tech Stack:** Roblox Studio, Luau (`--!strict`), Rojo 7.7.1, Rokit, Selene, StyLua, luau-lsp, Python 3 (CI scripts), Open Cloud (Luau Execution, place publishing), the built-in Studio MCP server, GitHub Actions.

**Spec:** `docs/spec/2026-11-13-bleak-game-design.md`. Platform facts: `docs/research/2026-11-13-roblox-platform-notes.md`.

This file is the contract between all workstreams. An agent working on one
workstream reads this file, the spec, and its own workstream plan.

## Global Constraints

- Every `.luau` file starts with `--!strict`.
- Rojo version is 7.7.1 or later. All other tools are pinned in `rokit.toml`.
- No third-party runtime packages. Utilities are written in `src/shared/Util`.
- Every gameplay number comes from `src/shared/Config`, which mirrors the spec. No literal gameplay numbers in services, AI or controllers.
- The server is authoritative. A client never decides status, items, stage, stun, grab or door state.
- State replicates through Attributes. One-off events use remotes (section 5).
- Every instance created during a match is parented under `Workspace.Runtime` and is destroyed on match reset.
- Nothing under `Workspace.Map` is created or destroyed at runtime. Doors and cages are reset by state.
- Player-facing text never names another game. Player-facing text is English and lives in `src/shared/Config/Strings.luau`.
- No blood, wounds or gore in any model, texture, decal, effect or sound.
- Before using an engine API, confirm it on `create.roblox.com/docs/reference`. Record any difference from the research notes in `docs/DECISIONS.md`.
- Every asset from the Creator Store has all `Script`, `LocalScript` and `ModuleScript` descendants removed before use, and its id is recorded in `assets/MANIFEST.md`.
- A task is not done until its tests pass, lint passes, and the reviewer approves.
- Commit messages use Conventional Commits and end with the attribution line required by the session.

## 1. Decisions

| Id | Decision | Reason |
|----|----------|--------|
| D-01 | Bleak is an original game in the genre, not a copy of the reference game | Spec section 2 |
| D-02 | One place. One server = one match. Lobby is an area of the same map | No teleport bugs. Testable in Studio |
| D-03 | State by Attributes, events by remotes | Late joiners and reconnects get state for free |
| D-04 | Rules are pure modules in `src/shared/Logic` and `src/server/AI/*Brain`. Services only move data between rules and instances | Rules are unit-testable where physics does not run |
| D-05 | The map is generated in Studio Edit mode by `tools/mapgen` from data files. Nobody edits the map by hand | Reproducible, reviewable, agent-friendly |
| D-06 | Classic `Humanoid` characters. Players own their character physics. The server owns all NPCs | Proven, smooth, least engine risk |
| D-07 | `StreamingEnabled` is off | The map is small. Removes a class of pathfinding and loading bugs |
| D-08 | The engine Server Authority feature is not used | Reported rough edges |
| D-09 | All input goes through `InputController` | The input backend can change in one file |
| D-10 | All sound uses the Audio API, not legacy `Sound` | Acoustic Simulation only works with it |
| D-11 | A carried player is shown by a visual proxy. The real character is hidden and anchored | Welding live characters to NPCs is a known bug source |
| D-12 | Tests use the in-repo `TestKit` (describe / it / expect) | Zero dependencies. Runs in Studio and in Open Cloud Luau Execution |
| D-13 | CI scripts are Python 3, standard library only | Same script on Windows and on GitHub runners |
| D-14 | Characters are spawned by the server (`CharacterAutoLoads` off) | The server controls where and when |

## 2. Human gates

An agent stops at a human gate, tells the user exactly what to do, and waits.
The step-by-step instructions for every gate, with a check for each step, are
in `docs/SETUP.md`. Before task F1 the Orchestrator asks the user to confirm
that parts A to C of that file are done, and runs the checks it can run
itself (`git --version`, `python --version`, `rokit --version`, and that the
three environment variables are set).

| Id | When | The user does |
|----|------|---------------|
| H-1 | Before F1 | Installs Roblox Studio and logs in. Installs Python 3.11+ if `python --version` fails |
| H-2 | Before F7 | Creates an experience named "Bleak" and a second experience "Bleak Test". Gives the universe ids and place ids |
| H-3 | Before F7 | Creates an Open Cloud API key with scopes `universe.place:write` and `universe.place.luau-execution-session:write` on "Bleak Test". Stores it as the `ROBLOX_API_KEY` user environment variable and as a GitHub Actions secret |
| H-4 | Before F8 | In Studio: enables the Studio MCP server (Assistant > … > Manage MCP Servers). Installs the Rojo Studio plugin. Runs `claude mcp add roblox-studio -- cmd.exe /c %LOCALAPPDATA%\Roblox\mcp.bat` |
| H-5 | Before E9 | In Studio Properties for `Lighting`: sets `LightingStyle` to `Realistic` and turns on `PrioritizeLightingQuality` |
| H-6 | Before A2 | Confirms ID verification status (audio upload quota) and approves the audio source list in `assets/MANIFEST.md` |
| H-7 | Each milestone end | In Studio: File > Save to File as `place/Bleak.rbxl`. Agent commits it |
| H-8 | Before Q12 | With 2–4 people, runs the multi-client checklist in `docs/qa/human-playtest.md` |
| H-9 | Release | Publishes from Studio (File > Publish to Roblox). Fills in the content maturity questionnaire. Sets Max Players to 4. Enables voice if wanted. Sets the experience public |

## 3. Repository layout

```
default.project.json        Rojo tree
rokit.toml  selene.toml  stylua.toml  .luaurc  .gitignore  .gitattributes
.github/workflows/ci.yml
src/
  shared/                   -> ReplicatedStorage.Shared
    Config/                 GameConfig, StageConfig, ItemConfig, WispConfig,
                            BrinelingConfig, DifficultyConfig, NoiseConfig,
                            PerkConfig, AudioConfig, Strings
    Types.luau
    Net/Remotes.luau
    Util/                   Signal, Rng, Clock, TableUtil, TestKit
    Logic/                  Stamina, LampCharge, InventoryRules, BrazierRules,
                            DetectionMath, MatchRules, KeyPlacement,
                            SpawnPlanner, PromptRules, XpRules, StageTable
  server/                   -> ServerScriptService.Server
    Main.server.luau
    Services/               one file per service (section 6)
    AI/                     Navigator, Perception, KeeperBrain, BrinelingBrain,
                            WispBrain, CarryProxy
  client/                   -> StarterPlayer.StarterPlayerScripts.Client
    Main.client.luau
    Controllers/            one file per controller
    UI/                     one file per screen or widget
  tests/                    -> ServerScriptService.Tests
    unit/*.spec.luau
    scenarios/*.scenario.luau
    RunUnit.luau  RunScenario.luau
tools/
  mapgen/                   MapBuild, Layout, Props, Materials, Markers, LightingRig
  ci/                       cloud_test.py, lint.ps1, lint.sh
assets/
  MANIFEST.md               every external asset id, source, licence note
place/Bleak.rbxl            milestone snapshot (binary)
docs/
  DECISIONS.md              deviations found during the build
  qa/                       bug reports, playtest logs, checklists
```

Instance tree at runtime:

```
Workspace
  Map            static, generated (Geometry, Props, Lights, Markers, Doors, Cages)
  Runtime        Items, Keeper, Brinelings, Wisps, Proxies, Effects
ReplicatedStorage
  Shared  Remotes  MatchState (Folder with attributes)  Assets
ServerStorage
  Prefabs        Items/<Kind>, Keeper, Brineling, Wisp, CarryProxy
```

## 4. Agents and ownership

| Agent | Plan | Owns (only this agent edits) |
|-------|------|------------------------------|
| Orchestrator | this file | `docs/superpowers/plans/*` checkboxes, merges, `docs/DECISIONS.md` |
| Foundation | 01 | root config files, `src/shared/Util`, `src/shared/Types.luau`, `src/shared/Net`, `src/shared/Config`, `tools/ci`, `.github` |
| Core | 02 | `src/shared/Logic` (except KeyPlacement, SpawnPlanner tests listed in 02), `src/server/Services/{Match,PlayerState,Lamp,Inventory,Item,Interaction,Brazier,Noise,Lobby,Data}Service.luau`, `src/server/Main.server.luau` |
| Physics | 03 | `src/server/Services/{Collision,Door,Zone,Recovery}Service.luau`, `src/server/Services/ItemPhysics.luau`, `src/server/AI/CarryProxy.luau`, `src/client/Controllers/{Movement,Camera}Controller.luau` |
| Monsters | 04 | `src/server/AI/*` except CarryProxy, `src/server/Services/{Keeper,Brineling,Wisp}Service.luau` |
| Environment | 05 | `tools/mapgen`, `assets/MANIFEST.md` (map and model rows), `src/client/Controllers/VfxController.luau` |
| Audio | 06 | `src/client/Controllers/AudioController.luau`, `src/client/Audio/*`, `src/shared/Config/AudioConfig.luau`, audio rows of `assets/MANIFEST.md` |
| UI | 07 | `src/client/UI/*`, `src/client/Controllers/{Input,Hud,Interaction,Inventory,Lamp,Lobby,Spectate,Settings}Controller.luau`, `src/client/Main.client.luau` |
| Game Tester | 08 | `src/tests/scenarios`, `src/tests/Run*.luau`, `src/server/Services/DebugService.luau`, `docs/qa` |
| Reviewer | — | Reads only. Approves or rejects each task |

Extra ownership, by exact file:

- Monsters: `src/client/Controllers/NpcVisualController.luau`, `src/shared/Config/AnimationIds.luau`, greybox prefab scripts `tools/mapgen/Prefabs/*Greybox.luau`.
- Environment: `src/client/Vfx/*`, all other `tools/mapgen/Prefabs/*`.
- UI: `src/shared/Config/InputConfig.luau`, `src/client/Controllers/{Viewmodel,Ending,Hint}Controller.luau`.
- Core: `PlayerStateService.registerFake` (Studio only), requested by the Game Tester in Q2.
- `AnimationIds.luau`, `InputConfig.luau` and `AudioConfig.luau` are the only `Config` files that change after M0.

Rules:

- An agent that needs a change in a file it does not own asks the
  Orchestrator, who assigns it to the owner as a task.
- `src/shared/Config` and `src/shared/Types.luau` are frozen after milestone
  M0. A change needs an entry in `docs/DECISIONS.md` and the Orchestrator's
  approval, because every workstream depends on them.
- Unit specs live in `src/tests/unit` and are written by the agent that owns
  the module under test.

## 5. Shared contracts

Foundation creates all of this in M0. Names here are exact.

### 5.1 Types (`src/shared/Types.luau`)

```lua
export type Difficulty = "Normal" | "Hard" | "Nightmare"
export type MatchPhase = "Lobby" | "Intro" | "Dormant" | "Active" | "Won" | "Lost" | "Results"
export type PlayerStatus = "Lobby" | "Alive" | "Carried" | "Downed" | "Spectating"
export type ItemKind = "OilCanister" | "VotiveCandle" | "FieldDressing" | "LampFlask"
	| "KeyCannery" | "KeyTower" | "KeyCellar" | "KeyBoathouse"
export type ZoneId = "Dock" | "Yard" | "HouseGround" | "HouseUpper" | "Attic" | "Cellar"
	| "Cannery" | "Boathouse" | "LampHall" | "Tower" | "Cove"
export type KeeperState = "Dormant" | "Roam" | "Investigate" | "Chase" | "Enraged"
	| "Carry" | "Stunned" | "Calm"
export type WispState = "Caged" | "Wander" | "Flee" | "Follow" | "Anchored" | "Released"
export type BrinelingState = "Emerging" | "Chase" | "Latched" | "Burning" | "Dead"
export type PerkId = "Fleetfoot" | "SteadyHands" | "DeepReservoir" | "TrimmedWick"
	| "FieldMedic" | "QuietSoles"
export type StageParams = {
	roamSpeed: number, chaseSpeed: number, enragedSpeed: number, carrySpeed: number,
	enrageMin: number?, enrageMax: number?, stunsToCalm: number, beamDwell: number,
	stunTime: number, calmTime: number, sightDelay: number, doorDelay: number,
	brinelingCap: number, brinelingInterval: number?, emberImmunity: number,
}
export type NoiseEvent = { position: Vector3, radius: number, kind: string, source: Player? }
export type BrazierState = { lit: boolean, charges: number, released: number }
```

### 5.2 Attributes

On each `Player`:

| Attribute | Type | Meaning |
|-----------|------|---------|
| `Status` | string | `PlayerStatus` |
| `Stamina` | number | Integer 0–100 |
| `Exhausted` | boolean | |
| `Sprinting`, `Crouched` | boolean | Server-accepted movement mode |
| `LampOn`, `BeamOn`, `LampLocked` | boolean | |
| `LampCharge` | number | Integer 0–125 |
| `HeldItem` | string | `ItemKind` or `""` |
| `CandleLit` | boolean | Held candle is lit |
| `CandleEndsAt` | number | Server time when the held candle burns out, or 0 |
| `LatchMeter` | number | Integer 0–100 (percent of 3.0 s) |
| `LastBreath` | number | Charges left |
| `Chased` | boolean | A Keeper is in `Chase` or `Enraged` on this player |
| `Ready`, `IsHost` | boolean | Lobby |

On `ReplicatedStorage.MatchState`: `Phase` (string), `Stage` (number),
`Released` (number), `Difficulty` (string), `BrazierLit` (boolean),
`BrazierCharges` (number), `PhaseEndsAt` (number, server time or 0),
`Solo` (boolean).

On runtime and map instances:

| Instance | Attributes |
|----------|------------|
| Keeper model | `State`, `Stage`, `Ember` |
| Brineling model | `State` |
| Wisp model | `State` |
| Door model | `Open`, `Locked` |
| Cage model | `Opened` |
| Item model | `Kind`, `Lit`, `LitEndsAt` |
| Sconce attachment's parent part | `CandleEndsAt` (0 when empty) |

Server time is always `Workspace:GetServerTimeNow()`.

### 5.3 Remotes (`ReplicatedStorage.Remotes`)

Client to server (`RemoteEvent` unless noted):

| Name | Arguments | Rate limit per player |
|------|-----------|-----------------------|
| `Interact` | `target: Instance, phase: "Begin" \| "Cancel"` | 10 / s |
| `DropItem` | — | 5 / s |
| `UseItem` | — | 5 / s |
| `SetLamp` | `on: boolean` | 10 / s |
| `SetBeam` | `on: boolean` | 10 / s |
| `SetMove` | `sprint: boolean, crouch: boolean` | 20 / s |
| `AimUpdate` (`UnreliableRemoteEvent`) | `look: Vector3` | 20 / s |
| `LobbyReady` | `ready: boolean` | 5 / s |
| `LobbyDifficulty` | `difficulty: string` | 5 / s |
| `LobbyStart` | — | 2 / s |
| `PerkBuy` | `perkId: string` | 2 / s |
| `PerkEquip` | `perkIds: {string}` | 2 / s |
| `SettingsSave` | `settings: {[string]: any}` | 1 / s |

Server to client (`RemoteEvent`):

| Name | Arguments |
|------|-----------|
| `Notify` | `code: string, args: {[string]: any}?` |
| `HoldProgress` | `target: Instance?, action: string?, startedAt: number, duration: number` (nil target = cancelled) |
| `Cue` | `cueId: string, position: Vector3?, source: Instance?` |
| `KeeperEvent` | `kind: "Wake" \| "Scream" \| "Grab" \| "GrabMiss" \| "Stunned" \| "Freed"` |
| `CarryView` | `proxy: Model?` (nil = carry ended) |
| `Results` | `{ won: boolean, released: number, difficulty: string, duration: number, xpGained: number, rank: number }` |
| `Profile` | `{ xp: number, rank: number, tokens: number, ownedPerks: {string}, equippedPerks: {string}, settings: {[string]: any} }` |

Every client-to-server handler validates argument types, the player's status,
and the rate limit before doing anything. An over-limit call is dropped
without an error.

### 5.4 Map tags

The map generator creates these. Code finds map objects only by tag.

| Tag | On | Required attributes / children |
|-----|----|--------------------------------|
| `LobbySpawn` | Part (×4) | — |
| `PlayerSpawn` | Part (×4) | — |
| `SafeNode` | Part | `Zone` |
| `Zone` | Part (invisible box) | `ZoneId`, `Floor` (number), `AlwaysOpen` (boolean) |
| `ItemSpawn` | Part | `PointId` (unique string), `Kinds` (comma-separated `ItemKind`), `Zone` |
| `Door` | Model | Attributes `DoorId`, `Zone`, `LockedBy` (`ItemKind` or `""`), `OpenAngle`. Children `Panel` (PrimaryPart), `Hinge` (Part), `InsideSide` (Attachment on the zone side) |
| `Brazier` | Model (×1) | Children `Bowl` (Part), `Sconce1`, `Sconce2`, `Sconce3` (Parts) |
| `WispCage` | Model (×2) | Attribute `CageId` (`"A"` or `"B"`). Children `Gate` (Part), `WispSpawn1`…`WispSpawn5` (Attachments) |
| `KeeperSpawn` | Part (×1) | — |
| `WanderNode` | Part | `For` (`"Keeper"`, `"Wisp"` or `"Both"`), `Zone` |
| `RetreatNode` | Part | `Zone` |
| `DropPoint` | Part | `Zone` |
| `MinionSpawn` | Part | `Zone` |
| `SeeThrough` | Any part | — |
| `LobbyGate` | Part (×1, invisible wall) | — . Collides during `Lobby` and `Results`; `CanCollide = false` in every other phase |
| `AmbienceZone` | Part (invisible box) | `AmbienceId` |
| `AmbienceEmitter` | Part | `CueId` |
| `CameraShot` | Part | `ShotId` (`"Win"` or `"Lose"`) |
| `FlickerLight` | Part with a light child | `Group` (string) |
| `Lightning` | Part with a `SurfaceLight` or `PointLight` | — |

Minimum counts are in plan 05, task E3, and are enforced by the map validator
(plan 08, task Q3).

### 5.5 Collision groups

`Default`, `Players`, `Npc`, `Wisp`, `Items`, `Doors`, `Proxy`, `Markers`.
The matrix is in plan 03, task P1.

## 6. Service interfaces

Every service is a ModuleScript that returns a table with `init()`, `start()`
and `onMatchReset()`. `Main.server.luau` calls `init()` on all services in the
order below, then `start()` in the same order. Signals are `Signal` objects.

```lua
-- ZoneService (Physics)
ZoneService.zoneOf(position: Vector3): ZoneId?
ZoneService.floorOf(position: Vector3): number
ZoneService.isZoneOpen(zone: ZoneId): boolean        -- always-open or unlocked
ZoneService.zoneOpened: Signal<ZoneId>

-- CollisionService (Physics)
CollisionService.setGroup(instance: Instance, group: string)

-- PlayerStateService (Core)
PlayerStateService.getStatus(player: Player): PlayerStatus
PlayerStateService.setStatus(player: Player, status: PlayerStatus, reason: string)
PlayerStateService.statusChanged: Signal<Player, PlayerStatus, PlayerStatus>   -- new, old
PlayerStateService.getAlive(): {Player}
PlayerStateService.getRoot(player: Player): BasePart?
PlayerStateService.getHeadPosition(player: Player): Vector3?
PlayerStateService.isMoving(player: Player): boolean
PlayerStateService.isSprinting(player: Player): boolean
PlayerStateService.isCrouched(player: Player): boolean
PlayerStateService.spawnCharacter(player: Player, at: CFrame)

-- MatchService (Core)
MatchService.getPhase(): MatchPhase
MatchService.getStage(): number
MatchService.getDifficulty(): Difficulty
MatchService.getStageParams(): StageParams
MatchService.isSolo(): boolean
MatchService.getRng(): Rng
MatchService.wake(reason: string)
MatchService.phaseChanged: Signal<MatchPhase, MatchPhase>    -- new, old
MatchService.stageChanged: Signal<number>

-- NoiseService (Core)
NoiseService.emit(position: Vector3, radius: number, kind: string, source: Player?)
NoiseService.noise: Signal<NoiseEvent>

-- LampService (Core)
LampService.isLampOn(player: Player): boolean
LampService.isBeamOn(player: Player): boolean
LampService.getBeam(player: Player): (Vector3, Vector3)?      -- origin, unit direction
LampService.refill(player: Player, amount: number)
LampService.registerTarget(id: string, getPosition: () -> Vector3?, onDwell: (player: Player, dt: number) -> ())
LampService.unregisterTarget(id: string)

-- ItemService (Core)
ItemService.spawnAll()
ItemService.spawnAt(kind: ItemKind, cframe: CFrame, state: {lit: boolean, litEndsAt: number}?): Model
ItemService.despawn(item: Model)
ItemService.count(kind: ItemKind): number                     -- world + held

-- InventoryService (Core)
InventoryService.getHeld(player: Player): ItemKind?
InventoryService.pickUp(player: Player, item: Model): boolean
InventoryService.drop(player: Player, reason: string)
InventoryService.consume(player: Player): ItemKind?
InventoryService.heldChanged: Signal<Player, ItemKind?>

-- InteractionService (Core)
InteractionService.register(target: Instance, handlers: {[string]: InteractionHandler})
InteractionService.unregister(target: Instance)
-- type InteractionHandler = {
--   canStart: (player: Player) -> (boolean, string?),   -- false, notifyCode
--   onComplete: (player: Player) -> (),
--   onTick: ((player: Player, dt: number) -> ())?,
-- }
-- The action name and hold time come from PromptRules.resolve (plan 02, C6).

-- DoorService (Physics)
DoorService.isOpen(door: Model): boolean
DoorService.isLocked(door: Model): boolean
DoorService.setOpen(door: Model, open: boolean, fromPosition: Vector3)
DoorService.unlock(door: Model)
DoorService.npcOpen(door: Model, fromPosition: Vector3, delay: number)
DoorService.doorFor(part: BasePart): Model?
DoorService.doorChanged: Signal<Model>

-- RecoveryService (Physics)
RecoveryService.nearestSafeNode(position: Vector3): CFrame

-- BrazierService (Core)
BrazierService.getState(): BrazierState
BrazierService.getBowlPosition(): Vector3
BrazierService.litSconcePositions(): {Vector3}
BrazierService.ignited: Signal<>
BrazierService.wispReleased: Signal<number>                   -- new released count

-- WispService (Monsters)
WispService.openCage(cageId: string)
WispService.nearestWisp(position: Vector3, radius: number): string?
WispService.release(wispId: string)
WispService.cageOpened: Signal<string>

-- KeeperService (Monsters)
KeeperService.getState(): KeeperState
KeeperService.getPosition(): Vector3?
KeeperService.getTarget(): Player?
KeeperService.stateChanged: Signal<KeeperState, KeeperState>

-- BrinelingService (Monsters)
BrinelingService.getCount(): number

-- CarryProxy (Physics)
CarryProxy.begin(victim: Player, keeper: Model): Model
CarryProxy.finish(victim: Player, dropCFrame: CFrame)

-- DataService (Core)
DataService.getProfile(player: Player): Profile?
DataService.hasPerk(player: Player, perk: PerkId): boolean
DataService.addXp(player: Player, amount: number)
DataService.buyPerk(player: Player, perk: PerkId): (boolean, string?)
DataService.equipPerks(player: Player, perks: {PerkId}): (boolean, string?)

-- LobbyService (Core)
LobbyService.getHost(): Player?

-- DebugService (Game Tester). Loaded only when RunService:IsStudio() is true.
-- Commands are listed in plan 08, task Q2.
```

Init order: Collision, Zone, Recovery, Data, PlayerState, Match, Noise, Lamp,
Item, Inventory, Interaction, Door, Brazier, Wisp, Keeper, Brineling, Lobby,
Debug.

Until a real service exists, its owner provides a stub with the same
interface that returns safe defaults. Foundation creates all stubs in task F6,
so every workstream compiles from the first day.

## 7. Standard task cycle

Every task in plans 01–08 follows these steps unless it says otherwise.

- [ ] 1. Read the task, its Interfaces block, and the spec sections it cites.
- [ ] 2. Write the failing tests listed in the task.
- [ ] 3. Run them. Confirm they fail for the expected reason.
- [ ] 4. Implement the smallest code that passes.
- [ ] 5. Run the task's tests, then the whole unit suite.
- [ ] 6. Run lint: `tools/ci/lint.ps1` (Windows) or `tools/ci/lint.sh`.
- [ ] 7. Commit with the message given in the task.
- [ ] 8. Request review. Fix findings. Mark the checkbox.

### Test commands

| Tier | What | Command |
|------|------|---------|
| T1 unit | Pure logic and brains | `python tools/ci/cloud_test.py --filter <name>` (headless), or MCP `execute_luau` in Edit context: `return require(game.ServerScriptService.Tests.RunUnit)("<name>")` |
| T2 scenario | Services in a running server | MCP `start_stop_play`, then `execute_luau` in Server context: `return require(game.ServerScriptService.Tests.RunScenario)("<name>")` |
| T3 agent playtest | Real input through Studio | MCP `character_navigation`, `user_keyboard_input`, `user_mouse_input`, `screen_capture`, `get_console_output` |
| T4 human playtest | 2–4 real clients | `docs/qa/human-playtest.md` |

Expected T1 output: a table `{ passed = N, failed = 0, failures = {} }`, and
`cloud_test.py` prints `PASS N/N` and exits with code 0.

Only one agent may use Studio at a time. The Orchestrator hands out the
**Studio token**. T1 through `cloud_test.py` needs no token and runs in
parallel.

## 8. Milestones

| Milestone | Content | Exit gate |
|-----------|---------|-----------|
| M0 Foundation | Plan 01, all tasks | CI green. Stubs compile. A player spawns on a flat baseplate |
| M1 Vertical slice | C1–C9, P1–P4, M1–M4 and M8, E1–E4, U1–U4, Q1–Q4 | In the greybox map, one player can light the Brazier, free one Wisp, be chased, stun the Keeper |
| M2 Full rules | C10–C13, P5–P7, M5–M7, M9–M11, U5–U8, Q5–Q8 | A full match can be won and lost on all three difficulties with 1 and with 2 players. Every FS rule has a passing scenario |
| M3 Art and audio | E5–E12, A1–A9, M12, U9 | Looks and sounds as spec 16. Performance budgets met in greybox are still met |
| M4 Meta and polish | U10–U13, C14, Q9–Q10 | Progression, settings, gamepad and touch work |
| M5 Hardening | Q11–Q14 | Zero open Blocker or Major bugs. Human playtest passed. Release checklist signed |

Dependency rules:

- Plan 01 finishes before anything else starts.
- Inside a milestone, the workstreams run in parallel.
- Task ids give order inside a workstream. A task may list extra
  dependencies in its own "Needs" line.
- The Game Tester runs the milestone gate. A milestone is closed only when
  the gate passes on `main`.

## 9. Orchestration protocol

For each task:

1. The Orchestrator checks the task's "Needs" are merged.
2. The Orchestrator starts one implementer subagent in its own git worktree on
   branch `ws/<task-id>-<slug>`.
3. The brief contains: the task text copied in full, the paths of the spec and
   this file, the relevant section numbers, the owned files, and the sentence
   "Do not edit files outside your ownership list. Ask instead."
4. The implementer follows the standard task cycle and reports: files
   changed, test output, and any deviation.
5. The Orchestrator starts a Reviewer subagent with the diff and the task
   text only. The Reviewer checks the task's tests exist and pass, the
   Interfaces match section 6 exactly, the Global Constraints hold, and the
   Review Focus items that the task owns.
6. On approval the Orchestrator merges to `main` with a merge commit, runs
   the unit suite on `main`, and ticks the checkbox.
7. On rejection the same implementer fixes the findings. After two
   rejections the Orchestrator re-reads the task and rewrites the brief.

Parallel limits: at most 4 implementers at once. At most 1 holds the Studio
token.

When an agent finds that the plan is wrong or incomplete, it stops, writes
the problem in `docs/DECISIONS.md` under "Open", and reports. The Orchestrator
decides, updates the plan file, and moves the entry to "Decided".

## 10. Bug loop

1. The Game Tester files `docs/qa/bugs/BUG-<nnn>.md` from the template in
   plan 08, task Q1.
2. Severity: Blocker (cannot finish a match, crash, data loss), Major (rule
   broken, softlock risk, exploit), Minor (wrong but playable), Polish.
3. The Orchestrator assigns the bug to the owner of the file at fault.
4. The owner first writes a failing T1 spec or T2 scenario that reproduces
   it, then fixes it, then the Game Tester re-runs the original steps.
5. A bug is closed only by the Game Tester.

## Review Focus

These are the conditions the spec implies that are most likely to hurt a
player. Each has a test in the task named.

1. **A player disconnects at the worst moment** (holding the last key, being
   carried, mid-hold, last one alive). Expected: FS-4 to FS-8. Tests: plan 08
   Q6, plan 02 C8 and C10.
2. **The character dies, resets or falls out of the world.** Expected: FS-3;
   status is kept; no orphan proxy or held item. Tests: plan 03 P7.
3. **Two players act on the same thing in the same tick** (same item, same
   door, same downed teammate, same Brazier). Expected: exactly one succeeds,
   no duplicated or lost item. Tests: plan 02 C5 and C7.
4. **An NPC cannot reach where it wants to go** (closed locked door, moved
   door, target in an unreachable spot). Expected: FS-9, FS-10; never frozen
   for more than the timeout. Tests: plan 04 M2 and M11.
5. **A client sends forged or spammed remotes** (interact with a far target,
   wrong types, 1,000 calls per second, beam while downed). Expected: dropped
   silently; no server error; no state change. Tests: plan 02 C2 and plan 08
   Q7.

## Risks

| Risk | Mitigation |
|------|------------|
| Open Cloud Luau Execution behaves differently from the research notes | F7 proves it first. Fallback: T1 runs through Studio `execute_luau` with the Studio token |
| `LightingStyle` cannot be set by any script | Human gate H-5 |
| Open Cloud publish drops SurfaceAppearance | Release is published from Studio (H-9). CI only publishes to "Bleak Test" for T1 |
| AI-generated or store assets look inconsistent | E5 fixes one art bible and one material library first; every asset is checked against it in E12 |
| Audio upload quota or moderation delays | A2 prefers Creator Store audio already cleared for use; uploads are a last resort and need H-6 |
| Server-owned NPC motion looks jerky | M12 adds client-side smoothing of the visual rig; gate measured in Q10 |
| Studio MCP input tools cannot drive two clients | Multi-client checks are T4 (H-8) |
| The spec numbers are untuned | Q13 is a tuning pass; changes go through `docs/DECISIONS.md` and the spec |

## Definition of done (release)

- All checkboxes in plans 01–08 are ticked.
- T1 suite passes in CI on `main`.
- Every T2 scenario in plan 08 passes.
- Zero open Blocker or Major bugs.
- Performance budgets (spec 16.3) are met and logged in `docs/qa/perf.md`.
- Human playtest (H-8) is signed in `docs/qa/human-playtest.md`.
- `assets/MANIFEST.md` lists every external asset.
- The release checklist (plan 08, task Q14) is complete.
