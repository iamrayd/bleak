# Bleak — game design spec

Version 1.0, 2026-11-13. This document is the single source of every gameplay
number. Plans refer to it by section (for example "spec 10.3"). When a number
must change, change it here first, then in `src/shared/Config`.

Units: distance in studs, time in seconds, speed in studs per second, angles
in degrees.

## 1. Pillars

1. **Progress feeds the threat.** Every soul freed makes the Keeper worse. The
   team decides when to push.
2. **Light is a tool and a risk.** The lamp shows the way and stuns, and it
   also makes the player visible.
3. **Nobody is out of the game.** A caught player is carried off and can be
   rescued.
4. **No softlocks.** No random placement, disconnect or physics accident may
   leave a match that cannot be won. Section 14 lists the rules.

## 2. Originality

Bleak is an original game. It shares genre mechanics with DEVOUR (see
`docs/research/2026-11-13-genre-reference.md`). It must not use that game's
name, characters, story, maps, models, textures, sounds or music. Nothing in
the repository or the published experience may name it in player-facing text.

## 3. Fiction (short)

Greyharbor Light is a lighthouse on a sea rock. Ten crew of the trawler
*Marrow* drowned within sight of it. Something from the deep, the Undertow,
holds their souls as Wisps and wears the last keeper as a puppet. The players
are a salvage party who light the old Beacon Brazier and release the ten
Wisps. Each release loosens the Undertow's grip and enrages the Keeper. After
the tenth, the Keeper is freed and the light comes back on.

## 4. Session

### 4.1 Players and servers

- 1–4 players. Server `MaxPlayers` = 4.
- One server runs one match at a time. There is no teleport between places.
- A player who joins during a match becomes `Spectating` and enters the lobby
  when the match ends.

### 4.2 Match phases

| Phase | Entered when | Notes |
|-------|--------------|-------|
| `Lobby` | Server start, or 15 s after `Results` begins | Players stand on the Dock. Ready-up |
| `Intro` | Host presses Start and all lobby players are ready | 5 s countdown, players moved to `PlayerSpawn` points |
| `Dormant` | Intro ends | Keeper sits in the Parlor. No Brinelings |
| `Active` | First wake trigger (10.2) | Normal play |
| `Won` | 10th Wisp released | 12 s ending sequence |
| `Lost` | `MatchRules.evaluate` returns `Lost` | 6 s ending sequence |
| `Results` | `Won` or `Lost` sequence ends | 15 s results screen, then `Lobby` |

- Host = the player who has been in the server longest. Host picks difficulty.
- Minimum 1 ready player to start.
- There is no match timer.

### 4.3 Win and lose

- Win: `released == 10`.
- Lose: no player has status `Alive` and no Last Breath is pending (13).
- Win is checked before lose in the same tick.

## 5. Player

### 5.1 Status

`Lobby`, `Alive`, `Carried`, `Downed`, `Spectating`.

### 5.2 Movement

| Value | Number |
|-------|--------|
| Walk speed | 10 |
| Sprint speed | 17 |
| Crouch speed | 5 |
| Downed crawl speed | 3 |
| Jump | Disabled (`JumpHeight` 0) |
| Stamina maximum | 100 |
| Sprint drain | 20 per second |
| Regen | 12 per second |
| Regen delay after sprint stops | 1.5 |
| Exhausted when | Stamina reaches 0 |
| Exhaustion clears at | Stamina ≥ 25 |

- Sprint needs: status `Alive`, moving, not crouched, not exhausted, not in a
  hold interaction.
- Camera: first person, locked. Field of view 70, sprint 76 (0.25 s tween).
- Head bob amplitude 0.08 walk, 0.14 sprint. A setting can turn it off.
- Characters take no damage. Health is not a mechanic.
- The Reset button is disabled.

### 5.3 Downed

- A downed player crawls, cannot sprint, cannot hold items, cannot use the lamp.
- A downed player drops their held item where they fall.
- Revive: a teammate holding a Field Dressing holds Interact for 4.0 s within
  7 studs. The dressing is consumed. The revived player is `Alive` with 50
  stamina and their previous lamp charge.
- Teammates see downed and carried players as an outline through walls.

## 6. Storm Lamp

Every player always has a lamp. It does not use the item slot.

| Value | Number |
|-------|--------|
| Lamp light range / angle / brightness | 60 / 70 / 2.2 (SpotLight) |
| Beam light range / angle / brightness | 40 / 22 / 6 |
| Beam hit cone half-angle | 9 |
| Beam hit range | 28 |
| Charge maximum | 100 |
| Beam drain | 14 per second |
| Regen | 2.5 per second |
| Regen delay after beam stops | 3.0 |
| Beam locks out at | Charge 0 |
| Lockout clears at | Charge ≥ 15 |
| Lamp Flask refill | +60, capped at 100 |

- Lamp on/off is a toggle. Beam is hold-to-use and needs the lamp on.
- The beam affects every Keeper and Brineling inside the cone with line of
  sight. Dwell time accumulates per target and decays at 1.0 per second when
  the target is not in the beam.
- On Nightmare the beam is disabled (12).

## 7. Items

### 7.1 Inventory

One held item. Picking up while holding swaps: the held item drops at the
player's feet.

### 7.2 Item kinds

| Kind | Use | Initial count | Candidate points | Respawn |
|------|-----|---------------|------------------|---------|
| `OilCanister` | Pour into Brazier (3.0 s) | 14 | 26 | None |
| `VotiveCandle` | Light (instant), lure Wisps, place on sconce (1.0 s) | 8 | 16 | 1 every 75 s while fewer than 4 exist |
| `FieldDressing` | Revive (4.0 s) | 4 | 10 | 1 every 120 s while fewer than 2 exist |
| `LampFlask` | Refill lamp (instant) | 5 | 12 | 1 every 100 s while fewer than 2 exist |
| `KeyCannery` | Unlock Cannery (1.5 s) | 1 | 6 | None |
| `KeyTower` | Unlock Tower Stair (1.5 s) | 1 | 6 | None |
| `KeyCellar` | Unlock Cellar (1.5 s) | 1 | 5 | None |
| `KeyBoathouse` | Unlock Boathouse (1.5 s) | 1 | 5 | None |

- "Exist" counts items in the world and in hands, not consumed ones.
- At least 6 Oil Canisters spawn in always-open zones.
- A key is consumed when used. The door stays unlocked for the match.
- Candle: once lit it burns for 120 s whether held, dropped or on a sconce,
  then it is destroyed. An unlit candle lasts forever.

### 7.3 Interaction rules

- Range 7, with line of sight from the camera to the target.
- Instant actions: pick up, drop, light candle, use Lamp Flask, open or close
  an unlocked door, toggle lamp.
- Hold actions: pour 3.0, release Wisp 2.5, revive 4.0, unlock 1.5, open cage
  2.0, place candle 1.0.
- A hold is cancelled when the player moves more than 2 studs from where it
  began, releases the key, changes status, or the target becomes invalid.
- Two players cannot hold the same target at once. The second gets "Busy".

## 8. World

### 8.1 Zones

| ZoneId | Floor | Always open | Locked by | Contains |
|--------|-------|-------------|-----------|----------|
| `Dock` | 0 | Yes | — | Lobby and player spawns |
| `Yard` | 0 | Yes | — | Paths between buildings, well house |
| `HouseGround` | 0 | Yes | — | Entry, Kitchen, Parlor (Keeper start), Study |
| `HouseUpper` | 1 | Yes | — | Landing, two bedrooms, bathroom |
| `Attic` | 2 | Yes | — | Storage, drop point |
| `Cellar` | −1 | No | `KeyCellar` | Storage, drop point |
| `Cannery` | 0 | No | `KeyCannery` | Work floor, Cold Room with Wisp Cage A |
| `Boathouse` | 0 | No | `KeyBoathouse` | Slipway, loft, drop point |
| `LampHall` | 0 | Yes | — | Tower base. The Brazier |
| `Tower` | 1–4 | No | `KeyTower` | Stair, Lantern Room with Wisp Cage B |
| `Cove` | 0 | Yes | — | Rocks below the house, drop point |

The island fits in a 420 × 300 footprint.

### 8.2 Key placement

| Key | May spawn in |
|-----|--------------|
| `KeyCannery` | HouseGround, HouseUpper, Attic, Yard |
| `KeyTower` | Cannery, HouseUpper, Attic, Cove |
| `KeyCellar` | HouseGround, Yard, Cannery |
| `KeyBoathouse` | Cove, Yard, HouseUpper |

Every placement must pass `KeyPlacement.isSolvable`: starting from the
always-open zones, repeatedly collecting reachable keys must open every zone.

### 8.3 Doors

- A door is open or closed, locked or unlocked.
- A closed door blocks sight, the beam and movement.
- Any door can always be opened from the inside of its zone, locked or not.
- A door swings away from the opener. Open time 0.45 s. Open angle 95.
- A door cannot close on a character: it waits until the swing volume is clear.
- The Keeper opens unlocked doors after a stage-based delay (10.3). It cannot
  pass locked doors. Brinelings open unlocked doors after 1.5 s. Wisps pass
  through all doors.

## 9. Ritual

### 9.1 Brazier

- State: `lit` (boolean), `charges` (0–3), `released` (0–10).
- Pouring a canister when unlit lights it. Charges stay 0.
- Pouring when lit adds 1 charge. Pouring is refused at 3 charges.
- Releasing needs: lit, charges ≥ 1, at least one Wisp within 5 studs of the
  bowl, and a 2.5 s hold on the bowl. It consumes 1 charge and releases the
  nearest Wisp. `released` rises by 1.
- 11 canisters are needed in total. 14 spawn.
- The Brazier has 3 sconces. Each holds one candle.

### 9.2 Wisp cages

Cage A (Cannery Cold Room) and Cage B (Tower Lantern Room) hold 5 Wisps each.
A 2.0 s hold opens a cage. The cage has no lock of its own; its zone's key
gates it.

### 9.3 Wisps

States: `Caged`, `Wander`, `Flee`, `Follow`, `Anchored`, `Released`.

| Value | Number |
|-------|--------|
| Wander speed | 6 |
| Flee speed | 13 |
| Follow speed | 9 |
| Flee trigger: player without a lit candle within | 10 |
| Flee trigger: Keeper within | 15 |
| Flee duration | 4.0, then `Wander` |
| Follow trigger: lit candle held within | 14 |
| Follow distance kept | 4 |
| Follow lost beyond | 20, or candle out, or holder not `Alive` |
| Anchor trigger: lit candle on a sconce within | 8 |
| Anchor leash from bowl | 5 |
| Stuck timeout | 10 s without progress → teleport to nearest Wisp wander node |

- Priority: `Anchored` > `Follow` > `Flee` > `Wander`.
- A Wisp follows the nearest lit candle holder.
- Wisps do not collide with characters. The Keeper ignores them.
- A Wisp is a hovering light 3 studs above the floor with a faint human shape.

## 10. The Keeper

### 10.1 States

`Dormant`, `Roam`, `Investigate`, `Chase`, `Enraged`, `Carry`, `Stunned`,
`Calm`.

### 10.2 Waking

`Dormant` ends on the first of:

- The Brazier is lit.
- A player stays within 8 studs of the Keeper for 2 s.
- A Wisp cage is opened.
- 240 s pass since `Dormant` began.

The Keeper plays a 4 s wake animation, then enters `Roam`. The phase becomes
`Active`.

### 10.3 Stage table (Normal)

Stage = `released` (0–9).

| Stage | Roam | Chase | Enrage interval min–max | Stuns to calm | Beam dwell | Stun time | Calm time | Sight delay | Door delay | Brineling cap | Brineling interval |
|-------|------|-------|-------------------------|---------------|-----------|-----------|-----------|-------------|-----------|---------------|--------------------|
| 0 | 7.0 | 13.5 | none | 1 | 1.0 | 4.0 | 20 | 0.40 | 1.2 | 0 | — |
| 1 | 7.5 | 14.0 | 110–140 | 1 | 1.0 | 4.0 | 20 | 0.40 | 1.2 | 1 | 60 |
| 2 | 8.0 | 14.5 | 100–130 | 1 | 1.0 | 4.0 | 18 | 0.40 | 1.2 | 1 | 55 |
| 3 | 8.5 | 15.0 | 90–120 | 1 | 1.1 | 3.6 | 18 | 0.40 | 1.0 | 2 | 50 |
| 4 | 9.0 | 15.5 | 80–105 | 2 | 1.2 | 3.2 | 16 | 0.40 | 1.0 | 2 | 45 |
| 5 | 9.5 | 16.0 | 70–95 | 2 | 1.3 | 3.0 | 14 | 0.25 | 0.8 | 3 | 40 |
| 6 | 10.0 | 16.5 | 60–85 | 2 | 1.4 | 2.8 | 12 | 0.25 | 0.6 | 3 | 35 |
| 7 | 10.5 | 17.0 | 50–70 | 2 | 1.5 | 2.6 | 10 | 0.25 | 0.5 | 4 | 30 |
| 8 | 11.0 | 17.5 | 40–60 | 3 | 1.6 | 2.3 | 9 | 0.25 | 0.4 | 5 | 26 |
| 9 | 11.5 | 18.0 | 30–45 | 3 | 1.8 | 2.0 | 8 | 0.25 | 0.3 | 6 | 22 |

- Enraged speed = Chase + 1.5.
- Carry speed = Roam + 2.
- Ember immunity: at stage ≥ 7, the Keeper ignores the beam for 3.0 s after a
  stun ends. Its eyes glow ember-orange during this time.

### 10.4 Perception

Vision:

- Cone half-angle 55, measured from the head's look direction.
- Range by target: beam on 60; lamp on 45; lamp off and moving 22; lamp off
  and crouched or still 10.
- Needs line of sight from the Keeper's head to the target's head or root
  part. Parts tagged `SeeThrough` do not block.
- The target must stay visible for the stage's sight delay before `Chase`.
- Proximity: any `Alive` player within 5 studs with line of sight is detected
  at once, any direction.

Hearing:

| Noise | Radius |
|-------|--------|
| Sprint step (every 0.35 s) | 30 |
| Walk step (every 0.5 s) | 12 |
| Crouch step | 0 |
| Door open or close | 18 |
| Door opened while sprinting | 28 |
| Item dropped | 14 |
| Oil pour (each second) | 20 |
| Revive (each second) | 10 |
| Brineling burned | 25 |
| Wisp released | Whole map |

- A noise is heard when distance ≤ radius × floorFactor. floorFactor is 1.0 on
  the same floor and 0.6 otherwise.
- A heard noise sends a roaming or investigating Keeper to `Investigate` at
  the noise position. It is ignored in every other state.
- Voice chat is never heard by the AI.

### 10.5 Behaviour

- `Roam`: walk between Keeper wander nodes. Prefer nodes not visited in the
  last 3 picks. Wait 2–5 s at each node.
- `Investigate`: go to the position, look around for 8 s, then `Roam`.
- `Chase`: pursue one target. Re-path every 0.25 s. If the target is not seen
  for 6 s, `Investigate` at the last seen position.
- Random enrage: while `Active` and in `Roam` or `Investigate`, a timer drawn
  from the stage's interval fires `Enraged` on the nearest `Alive` player. The
  timer resets after every `Calm`.
- Release rage: on a Wisp release the Keeper stops, screams for 2.5 s
  (cannot move, can be stunned), then becomes `Enraged` on the `Alive` player
  nearest the Brazier. Releases during the scream do not restart it. The new
  stage's values apply at once.
- `Enraged`: like `Chase`, but always knows the target's position. It needs
  the stage's "stuns to calm" count. It ends when that count is reached, when
  the target is caught, or after 25 s with no `Alive` player within sight
  rules, which leads to `Roam`.
- `Chase` when not enraged needs 1 stun at every stage.
- `Stunned`: stands still for the stage's stun time. Then: if enraged and
  stuns remain, back to `Enraged`; otherwise `Calm`.
- `Calm`: walk to the retreat node farthest from all players. No detection for
  the stage's calm time. Then `Roam`.
- Retarget: in `Chase` or `Enraged`, if the target stops being `Alive`, pick
  the nearest visible `Alive` player, else `Investigate`.

### 10.6 Grab and carry

- Grab range 4. Wind-up 0.35 s. During the wind-up, if the target is beyond
  5.5 or sight breaks, the grab misses and the Keeper recovers for 1.0 s.
- On a hit the target becomes `Carried`. The target's real character is hidden
  and a visual proxy rides on the Keeper's shoulder (plan 03, task P6). The
  victim's camera follows the proxy.
- The Keeper walks to a drop point chosen at random from those that are in an
  unlocked or always-open zone and at least 40 studs from the Brazier.
- At the drop point the victim becomes `Downed` there. The Keeper enters
  `Calm` for half the stage's calm time.
- A stun during `Carry` drops the victim as `Downed` at the Keeper's position.
- If the path to the drop point fails for 5 s, the victim is dropped where the
  Keeper stands.

## 11. Brinelings

States: `Emerging`, `Chase`, `Latched`, `Burning`, `Dead`.

| Value | Number |
|-------|--------|
| Speed | 11 |
| Emerging time | 2.0 |
| Spawn rule | A `MinionSpawn` point ≥ 35 studs from every `Alive` player with no line of sight to any |
| Latch range | 3 |
| Latch meter gain | 1.0 per second in range with line of sight |
| Latch meter decay | 0.5 per second otherwise |
| Player downed at | Meter 3.0 |
| Beam dwell to burn | 0.6 |
| Burn time before removal | 0.8 |
| Stuck timeout | 8 s without progress → removed, not counted as burned |

- Cap and spawn interval come from the stage table.
- One meter per player. Several Brinelings in range do not stack the gain.
- A Brineling targets the nearest `Alive` player and re-targets every 1.0 s.
- After downing a player it targets the next. It ignores downed players.
- There are no Brinelings during `Dormant` or at stage 0.

## 12. Difficulty

| Modifier | Normal | Hard | Nightmare |
|----------|--------|------|-----------|
| Keeper speeds | ×1.00 | ×1.08 | ×1.08 |
| Stun time | ×1.00 | ×0.75 | No stun |
| Beam | Yes | Yes | Disabled |
| Item respawn intervals | ×1.0 | ×1.5 | ×1.5; Field Dressing never respawns |
| Brineling speed | 11 | 12 | 12 |
| Brineling lifetime | Unlimited | Unlimited | 45 s, then removed |
| Enrage end | By stuns | By stuns | Target caught, or 25 s |
| Last Breath charges (solo) | 2 | 1 | 0 |
| XP multiplier | 1.0 | 1.5 | 2.5 |

On Nightmare the lamp still lights the way. The Beam input does nothing and
the HUD hides the charge meter.

## 13. Solo

When a match starts with one player, that player has Last Breath charges
(table above). When downed with a charge left, the player revives alone after
15 s with 50 stamina and one charge is spent. The lose check waits while a
Last Breath is pending. If a second player was present at match start, Last
Breath is off for the whole match.

## 14. Fail-safes

Each rule has a test in plan 08.

| # | Situation | Rule |
|---|-----------|------|
| FS-1 | Key placement | Must pass `KeyPlacement.isSolvable` (8.2) |
| FS-2 | Item falls below Y = −40 or leaves the map bounds | Item returns to its spawn point |
| FS-3 | Player falls below Y = −40 or the character dies | Character respawns at the nearest `SafeNode`, keeping status |
| FS-4 | Player leaves while holding an item | Item drops at the last character position |
| FS-5 | Player leaves while carried | Proxy removed, Keeper enters `Calm` |
| FS-6 | Player leaves during a hold interaction | Hold cancelled, target freed |
| FS-7 | Last `Alive` player leaves | Lose check runs at once |
| FS-8 | All players leave | Match resets to `Lobby` |
| FS-9 | Keeper cannot path for 8 s | Keeper teleports to the nearest Keeper wander node out of every player's sight |
| FS-10 | Wisp or Brineling stuck | See 9.3 and 11 |
| FS-11 | Drop point in a locked zone | Never chosen (10.6) |
| FS-12 | Player downed inside a locked zone | Doors open from inside (8.3) |
| FS-13 | Fewer than 11 canisters could remain | Canisters cannot be destroyed except by pouring; FS-2 returns lost ones |
| FS-14 | Candle supply runs out | Candles respawn (7.2) |
| FS-15 | Host leaves in `Lobby` | Next-longest player becomes host |
| FS-16 | Player joins mid-match | `Spectating` (4.1) |
| FS-17 | Wisp inside a closed locked room | Wisps pass through doors (8.3) |

## 15. Progression

- XP on win: 100 × difficulty multiplier + 10 × released. On loss: 10 × released.
- Rank `r` is reached at XP ≥ 150 × r × (r + 1) / 2.
- 1 token per rank. Perks cost 2 tokens. Two perks equipped at most.

| Perk id | Effect |
|---------|--------|
| `Fleetfoot` | Sprint speed ×1.15 while a Keeper is in `Chase` or `Enraged` on you |
| `SteadyHands` | Hold interactions take ×0.70 time |
| `DeepReservoir` | Lamp charge maximum 125 |
| `TrimmedWick` | Lit candle lasts 160 s |
| `FieldMedic` | Revive hold takes 2.0 s |
| `QuietSoles` | Noise radii ×0.75 |

Saved data per player: `xp`, `ownedPerks`, `equippedPerks`, `settings`.
There are no purchases with real money in version 1.

## 16. Presentation targets

### 16.1 Look

- Realistic lighting style with quality prioritised. Night, storm, heavy fog.
- Every visible surface uses a PBR material (colour, normal, roughness,
  metalness). No default plastic.
- Interiors are lit only by the players' lamps, the Brazier, lightning and a
  few weak practical lights that flicker.
- No blood, wounds or gore anywhere.

### 16.2 Feel

- Input to camera response within one frame.
- No visible teleporting of NPCs during normal movement.
- All UI transitions ≤ 0.25 s.

### 16.3 Performance budgets

| Budget | Limit |
|--------|-------|
| Client frame time, graphics level 7, 4 players, 6 Brinelings | ≤ 16.7 ms on the test PC |
| Server heartbeat | ≥ 58 Hz |
| Instances in `Workspace` | ≤ 25,000 |
| Triangles in view | ≤ 600,000 |
| Shadow-casting lights in view | ≤ 6 |
| Texture resolution | ≤ 1024 × 1024 per map |
| Client memory | ≤ 1.6 GB |
| Network receive per client | ≤ 50 KB/s |
| Join to lobby interactive | ≤ 12 s on the test PC |

## 17. Platforms and compliance

- Keyboard and mouse, and gamepad, are fully supported. Touch is supported
  with on-screen controls.
- Target content maturity: Mild fear. Jump scares, a creepy NPC and ominous
  music are present. Blood is absent.
- Proximity voice chat is optional and enabled in Experience Settings.
- A photosensitivity setting removes lightning flashes and light flicker.

## 18. Out of scope for version 1

More than one map. Purchases. Cosmetics. Matchmaking across servers. Player
versus player. Console certification work.
