# Bleak Plan 05 — Map, Environment and Graphics

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Greyharbor Light: one original map that is fully playable in greybox, then dressed to a realistic, storm-night look inside the performance budgets.

**Architecture:** The map is data. `tools/mapgen` scripts run in Studio Edit mode and rebuild `Workspace.Map` from Luau data files. Gameplay code finds everything by tag (master plan 5.4). Art replaces greybox by changing data, never by hand edits.

**Tech Stack:** Studio MCP (`execute_luau`, `insert_asset`, `search_asset`, `generate_mesh`, `generate_material`, `screen_capture`), Terrain, SurfaceAppearance, MaterialVariant, Atmosphere, post-processing.

**Spec:** `docs/spec/2026-11-13-bleak-game-design.md` sections 8, 16, 17. Physics principles: plan 03, top.

Agent: **Environment**. Every task needs the Studio token. Every task uses the
standard task cycle (master plan section 7); "tests" here are the map
validator (plan 08, Q3), screenshots and measurements.

How mapgen runs: with `rojo serve` connected, `tools/mapgen` is **not** in
the Rojo tree. The agent sends the module sources to Studio with MCP
`execute_luau` (Edit context) through one loader script, `tools/mapgen/run.luau`,
which is pasted as the script body and which builds everything from the data
tables embedded by a small bundler, `tools/mapgen/bundle.py` (concatenates the
modules into one chunk with a local `require` table). Task E1 builds this.

---

### Task E1: Mapgen framework

**Files:**
- Create: `tools/mapgen/bundle.py`, `tools/mapgen/MapBuild.luau`, `tools/mapgen/Primitives.luau`, `tools/mapgen/README.md`

**Interfaces — Produces:**

```lua
MapBuild.run(options: { layers: {string}? })   -- layers: "Terrain","Geometry","Doors","Markers",
                                               -- "Props","Lights","Lighting","Prefabs"; nil = all
-- Each layer deletes Workspace.Map.<Layer> (or its own outputs) and rebuilds it. Idempotent:
-- running twice gives the same instance count and the same positions.

Primitives.floor(parent, name, centre: Vector3, sizeXZ: Vector2, thickness: number, material: string)
Primitives.wall(parent, name, a: Vector3, b: Vector3, height: number, thickness: number,
  material: string, openings: { { at: number, width: number, height: number, sill: number } }?)
  -- builds the wall as plain parts around the openings. No unions (CSG).
Primitives.ramp(parent, name, bottom: Vector3, top: Vector3, width: number, material: string)
Primitives.stairsVisual(parent, name, bottom: Vector3, top: Vector3, width: number, steps: number)
  -- visual steps with CanCollide = false; always paired with a ramp
Primitives.marker(parent, tag: string, cframe: CFrame, size: Vector3, attributes: {[string]: any})
Primitives.material(name: string): { Material: Enum.Material, Color: Color3, Variant: string? }
```

- `bundle.py <entry> -o build/mapgen.luau` produces one Luau chunk. `README.md` gives the exact loop: bundle, send with `execute_luau`, read the returned summary `{ parts = N, markers = N, warnings = {...} }`.
- Every created part: `Anchored = true`, `CastShadow` true for geometry, material from `Primitives.material`.
- A second run must report the same counts. `MapBuild.run` ends by calling `ChangeHistoryService:SetWaypoint("mapgen")`.

- [ ] **Step 1:** Write the files. Build a 20 × 20 test room with one door opening and one ramp.
- [ ] **Step 2:** Run twice. Expected: identical summaries; `screen_capture` shows the room.
- [ ] **Step 3:** Commit: `feat: add map generator framework`.

---

### Task E2: Greybox layout

**Files:**
- Create: `tools/mapgen/Layout.luau`, `tools/mapgen/Terrain.luau`

Coordinates: X east, Z south, Y up. Island centre (0, 0, 0). Storey height 14
(12 clear + 2 slab). Wall thickness 1. Door openings 5 × 8. Corridors ≥ 6 wide.

| Zone | Centre (X, Z) | Size (X × Z) | Floor Y | Rooms and links |
|------|---------------|--------------|---------|-----------------|
| `Dock` | (−170, 90) | 60 × 40 | 2 | Pier, moored boat. One `LobbyGate` across the pier's land end |
| `Yard` | (0, 20) | 220 × 140 | 6 | Open ground, well house (12 × 12) at (20, 30), stone paths to every building |
| `HouseGround` | (−60, −50) | 80 × 60 | 6 | Entry (20 × 20), Kitchen (30 × 30), Parlor (30 × 30), Study (20 × 24), hall loop 6 wide. Front door to Yard (south). East door to the covered walkway. Stair up. Cellar door (locked) |
| `HouseUpper` | (−60, −50) | 80 × 60 | 20 | Landing, Bedroom A (30 × 26), Bedroom B (26 × 26), Bathroom (16 × 14), second stair up to the Attic |
| `Attic` | (−60, −50) | 70 × 40 | 34 | One room with roof beams, stored crates |
| `Cellar` | (−60, −50) | 60 × 40 | −8 | Two rooms. Inside door from the house hall and an outside hatch door to the Cove path, both `LockedBy = KeyCellar` |
| `Cannery` | (90, 60) | 90 × 60 | 6 | Work floor with machine rows (cover), office (16 × 16), Cold Room (30 × 24) with Cage A. Two outside doors, both `KeyCannery`. Cold Room door unlocked |
| `Boathouse` | (−120, 110) | 50 × 36 | 3 | Slipway floor, loft at Y 15 by ramp stair. One door, `KeyBoathouse` |
| `LampHall` | (110, −70) | radius 22 | 6 | 16-sided room. Brazier in the centre. Door to Yard (south-west), door to the covered walkway (west), gate to the Tower stair (`KeyTower`) |
| `Tower` | (110, −70) | radius 22 → 16 | 6 → 62 | Spiral ramp 6 wide against the wall, slope ≤ 30°, four landings. Lantern Room (radius 16) at Y 62 with Cage B and glass walls tagged `SeeThrough` |
| `Cove` | (−40, −130) | 120 × 50 | 1 | Wet rocks, tide pools. Path ramps up to the Yard (west side) and to the Cellar hatch |

Covered walkway: 6 wide, from the house east door (−20, −50) to the LampHall
west door (88, −70), roofed, open sides with posts.

Loops required (so a chase always has two ways out): house hall loop; house
front door ↔ east door through the Yard; Cannery two doors; LampHall two open
doors; Yard ↔ Cove ↔ Cellar ↔ house (after the Cellar is unlocked).

Dead ends allowed (risk rooms): Attic, Lantern Room, Boathouse loft, Cold Room.

`Terrain.luau`: island body by `Terrain:FillBlock` and `FillBall` (Rock under,
Ground and Mud on top), flat at Y 6 under the Yard and buildings, sloping to
Y 1 at the Cove and Y 2 at the Dock. Sea: `Terrain:FillBlock` Water from Y −30
to Y 0 in a 2,048 × 2,048 square. Everything outside the island is water.

- [ ] **Step 1:** Write `Layout.luau` as data (rooms, walls with openings, floors, ceilings, ramps) and the builder that calls `Primitives`. Every room has a ceiling.
- [ ] **Step 2:** Build. `screen_capture` from above with ceilings hidden (set `Transparency` temporarily in the capture script, then restore) and from each zone's interior.
- [ ] **Step 3:** Check by script: no ramp steeper than 35°; no doorway narrower than 5 or lower than 8; no corridor narrower than 6 (measured from the layout data); zero unanchored parts.
- [ ] **Step 4:** Commit: `feat: add greybox layout`.

---

### Task E3: Doors and markers

**Files:**
- Create: `tools/mapgen/Doors.luau`, `tools/mapgen/Markers.luau`

**Doors:** one `Door` model per doorway in the table above and per interior
room door (target: 22 doors). `Zone` and `LockedBy` per the table. Interior
room doors in the house are unlocked, `LockedBy = ""`, `Zone` = their zone.
Children as master plan 5.4. `InsideSide` is on the zone's side.

**Markers — minimum counts:**

| Tag | Count | Placement rules |
|-----|-------|-----------------|
| `LobbySpawn` | 4 | On the pier |
| `PlayerSpawn` | 4 | On the pier's land end, past the `LobbyGate` |
| `LobbyGate` | 1 | Invisible wall across the pier, 20 × 12 × 1 |
| `SafeNode` | ≥ 12 | At least 1 per zone, on open floor |
| `Zone` | ≥ 11 | Boxes cover every walkable point. Each storey of the Tower has its own box with its own `Floor` |
| `ItemSpawn` (Oil) | 26 | ≥ 10 in always-open zones; ≥ 2 in each locked zone |
| `ItemSpawn` (Candle) | 16 | ≥ 8 in always-open zones |
| `ItemSpawn` (Dressing) | 10 | ≥ 6 in always-open zones |
| `ItemSpawn` (Flask) | 12 | ≥ 6 in always-open zones |
| `ItemSpawn` (keys) | 6 + 6 + 5 + 5 | Zones exactly as spec 8.2; each allowed zone has ≥ 1 point |
| `Brazier` | 1 | LampHall centre |
| `WispCage` | 2 | `A` in the Cold Room, `B` in the Lantern Room |
| `KeeperSpawn` | 1 | Parlor, on a chair facing a radio set |
| `WanderNode` (Keeper or Both) | ≥ 26 | ≥ 2 per zone, including locked zones |
| `WanderNode` (Wisp or Both) | ≥ 18 | In Yard, LampHall, Cannery, Tower, HouseGround |
| `RetreatNode` | ≥ 4 | Always-open zones only: Attic, Cove, HouseUpper, far Yard corner |
| `DropPoint` | 7 | Attic, Cove, HouseUpper bathroom, Cellar, Boathouse loft, Lantern Room, Cold Room |
| `MinionSpawn` | ≥ 18 | At floor drains, tide pools, the well; ≥ 1 per zone except Dock |
| `AmbienceZone` | ≥ 11 | One per zone; `AmbienceId` = the zone id |
| `AmbienceEmitter` | ≥ 10 | Dripping pipes, banging shutter, creaking beams, buoy bell, Cove waves; `CueId` from plan 06 |
| `CameraShot` | 2 | `Win`: in the Yard looking at the tower. `Lose`: in the Parlor looking at the chair |
| `FlickerLight` | ≥ 12 | Weak practical lights indoors |
| `Lightning` | ≥ 6 | Outside windows of the house, Cannery and Lantern Room |

Each `ItemSpawn` allows exactly one kind, sits on a surface (table, shelf,
floor, crate), and is reachable: within 7 studs of walkable floor with line
of sight. `PointId` is `<Zone>_<Kind>_<nn>`.

- [ ] **Step 1:** Write both files as data plus builders.
- [ ] **Step 2:** Build. Run the map validator (Q3). Expected: zero errors.
- [ ] **Step 3:** Commit: `feat: add doors and gameplay markers`.

---

### Task E4: Navigation check and greybox lighting

**Files:**
- Create: `tools/mapgen/NavCheck.luau`, `tools/mapgen/LightingRig.luau` (greybox values)

- [ ] **Step 1:** `NavCheck` (Edit context): with agent radius 2 and height 7.5, `ComputeAsync` from `KeeperSpawn` to every `WanderNode`, `DropPoint`, `RetreatNode`, `MinionSpawn`, `SafeNode` and the Brazier, with all doors set open. With radius 1 and height 4, from each cage to the Brazier. Expected: every path `Success`. Print failures with positions.
- [ ] **Step 2:** In Studio turn on navigation mesh display (ask the user if it cannot be set by script) and `screen_capture` each zone. Fix layout data where the mesh is broken (narrow gaps, steep ramps, door frames).
- [ ] **Step 3:** Greybox lighting so the slice is playable: `ClockTime = 0.5`, `Brightness = 0.6`, `Ambient = (10, 10, 12)`, `OutdoorAmbient = (20, 24, 32)`, `GlobalShadows = true`, one dim `PointLight` per room (range 16, brightness 0.4).
- [ ] **Step 4:** Commit: `feat: add nav check and greybox lighting`.

**End of M1 work for Environment.** M2 has no Environment tasks: fix map bugs
the Game Tester files, and start E5 early.

---

### Task E5: Art bible and material library

**Files:**
- Create: `docs/art/art-bible.md`, `tools/mapgen/Materials.luau`
- Modify: `assets/MANIFEST.md`

**Art bible content (write these decisions down):**
- Setting: North Atlantic rock, 1950s equipment, abandoned about thirty years. Wet, salt-stained, rust, peeling paint, rot. No modern objects.
- Palette: cold blue-greys and desaturated greens. Warm colour appears only in fire, lamp light, brass and the Keeper's ember eyes.
- Scale: 1 stud ≈ 0.28 m. Door 5 × 8, table height 3, chair seat 1.8, counter 3.5.
- Texel density: 1024 px per 16 studs on walls and floors.
- Six reference screenshots are produced in E9 and stored in `docs/art/reference/`. All later art is compared with them.
- Forbidden: blood, bodies, gore, religious symbols of real faiths, text or logos from real brands, anything recognisable from another game.

**Material library (names are exact; each has colour, normal, roughness, metalness):**
`WoodPlankWeathered`, `ClapboardPeeling`, `PlasterStained`, `StoneWet`, `ConcreteRough`, `MetalCorrugatedRust`, `IronDark`, `BrassTarnished`, `TileColdRoom`, `RockCoastal`, `MudWet`, `GrassDead`, `RopeHemp`, `GlassDirty`, `FabricOilskin`, `WoodFurnitureDark`.

Sourcing order per material: (1) MCP `generate_material`; (2) a Creator Store
PBR material pack; (3) the built-in material nearest in kind with a tuned
colour. Every material becomes a `MaterialVariant` in `MaterialService` and a
row in `Materials.luau` and in the manifest.

- [ ] **Step 1:** Write the art bible.
- [ ] **Step 2:** Produce the 16 materials. Build a test wall of 16 panels. `screen_capture` under the E4 lighting and under a lamp-only setup.
- [ ] **Step 3:** Check: tiling is not visible at 30 studs; normals are not inverted (light from the left brightens left-facing bumps); wet materials have roughness below 0.4.
- [ ] **Step 4:** Commit: `feat: add art bible and material library`.

---

### Task E6: Architecture art pass

**Files:** Modify `tools/mapgen/Layout.luau`, `Primitives.luau`. Create `tools/mapgen/Trim.luau`.

- Assign library materials to every wall, floor, ceiling and ramp by zone.
- `Trim.luau` adds modular detail from data: skirting boards, door frames, window frames with `GlassDirty` panes, ceiling beams, roof shapes (pitched house roof, Cannery sawtooth roof, Tower gallery rail), gutters, chimney, porch.
- Break up flat walls: every interior wall longer than 20 studs gets at least one of pilaster, shelf alcove, pipe run or window.
- Stairs get their visual steps and handrails. Collision stays on the ramps.
- Windows are openings filled with a `SeeThrough` glass part. No window is large enough to walk through (sill ≥ 4, or bars).

- [ ] **Step 1:** Implement. Rebuild `Geometry`.
- [ ] **Step 2:** Re-run Q3 and `NavCheck`. Expected: zero errors (art must not change navigation).
- [ ] **Step 3:** `screen_capture` every room from two corners. Check against the art bible.
- [ ] **Step 4:** Commit: `feat: add architecture art pass`.

---

### Task E7: Props pass

**Files:** Create `tools/mapgen/Props.luau` (data: `{ prefab, cframe, zone }`), `tools/mapgen/PropLibrary.luau`. Modify `assets/MANIFEST.md`.

**Prop list per zone (minimum):**
- House: radio set, armchairs, dining table and chairs, stove, sink, cupboards, bookshelves, desk, beds, wardrobes, bathtub, mirrors (non-reflective dirty glass), rugs, picture frames (abstract seascapes, no faces), crates, oil lamps.
- Cannery: conveyor tables, canning machines, stacked crates, barrels, hooks on rails (empty), scales, lockers, cold-room shelving, floor drains.
- Boathouse: rowing boat on the slip, winch, nets, oars, buoys, fuel drums.
- LampHall and Tower: spiral rail, lens mechanism, weights and chains, tool chest, logbook desk.
- Yard and Cove: well, washing line, lobster pots, anchor, fence, dead shrubs, driftwood, tide pools.
- Dock: bollards, ropes, the players' boat.

Sourcing order per prop: (1) Creator Store `search_asset` for a realistic PBR
mesh; (2) MCP `generate_mesh`; (3) built from parts. Rules for every prop:
remove all scripts; one material slot; ≤ 20,000 triangles (most ≤ 3,000);
re-texture with library materials when the look does not match; `Anchored`;
`CanCollide` only when larger than 2 studs and not in a walk path;
`CanQuery = false` for small clutter. Record every asset in the manifest.

Placement rules: no prop within 3 studs of a door swing; no prop narrows a
walk path below 6; every `ItemSpawn` stays reachable; cover objects in the
Cannery leave lanes ≥ 6 wide.

- [ ] **Step 1:** Build `PropLibrary` (prefabs stored in `ServerStorage.PropLibrary`, cloned by `Props`).
- [ ] **Step 2:** Place props zone by zone. After each zone: rebuild, `screen_capture`, run Q3 and `NavCheck`.
- [ ] **Step 3:** Commit after each zone: `feat: dress <zone>`.

---

### Task E8: Terrain, sea, sky and weather

**Files:** Modify `tools/mapgen/Terrain.luau`. Create `tools/mapgen/Sky.luau`. Environment-owned client code for weather goes in `VfxController` (E11).

- Terrain materials: `Rock`, `Ground`, `Mud`, `Sand` at the Cove. `Terrain.WaterWaveSize = 0.6`, `WaterWaveSpeed = 22`, `WaterColor = (18, 32, 38)`, `WaterTransparency = 0.25`, `WaterReflectance = 0.6`. `Terrain.Decoration = true` (grass) only on `Grass`; the island uses dead-grass colouring.
- Rocks: 30–40 large rock meshes around the shoreline to hide the terrain edge.
- Sky: a night `Sky` with heavy cloud, `StarCount = 0`, `CelestialBodiesShown = false`. `Clouds` under Terrain: `Cover = 0.9`, `Density = 0.8`, `Color = (40, 46, 56)`.
- Distant fog bank: `Atmosphere` handles it (E9).
- Invisible sea barrier: a ring of `Default` collision walls 12 high just off the shore so characters cannot wade away. Recovery (FS-3) is the backstop.

- [ ] **Step 1:** Implement and rebuild.
- [ ] **Step 2:** Walk the shoreline with `character_navigation`. Expected: no way off the island; no terrain hole.
- [ ] **Step 3:** Commit: `feat: add terrain, sea and sky`.

---

### Task E9: Final lighting and post-processing

**Needs:** H-5.

**Files:** Modify `tools/mapgen/LightingRig.luau`. Create `docs/art/reference/*.png`.

Starting values (tune by eye, then write the final values into `LightingRig.luau`):

| Object | Property | Value |
|--------|----------|-------|
| Lighting | `ClockTime` | 0.5 |
| Lighting | `Brightness` | 0.5 |
| Lighting | `Ambient` | (0, 0, 0) |
| Lighting | `OutdoorAmbient` | (16, 20, 28) |
| Lighting | `EnvironmentDiffuseScale` | 0.25 |
| Lighting | `EnvironmentSpecularScale` | 0.6 |
| Lighting | `ExposureCompensation` | −0.3 |
| Lighting | `ShadowSoftness` | 0.3 |
| Lighting | `GlobalShadows` | true |
| Atmosphere | `Density` / `Offset` / `Haze` / `Glare` | 0.42 / 0.1 / 2.2 / 0 |
| Atmosphere | `Color` / `Decay` | (70, 82, 96) / (30, 36, 44) |
| ColorCorrection | `Brightness` / `Contrast` / `Saturation` | −0.03 / 0.12 / −0.25 |
| ColorCorrection | `TintColor` | (225, 235, 255) |
| Bloom | `Intensity` / `Size` / `Threshold` | 0.5 / 28 / 1.6 |
| DepthOfField | `FarIntensity` / `FocusDistance` / `InFocusRadius` / `NearIntensity` | 0.15 / 40 / 30 / 0 |

Practical lights: `FlickerLight` parts hold a weak warm `PointLight`
(range 14, brightness 0.5, colour (255, 190, 130), shadows off). The Brazier
light: range 40, brightness 3 when lit, shadows on. Lantern Room lens light
comes on only in the `Won` ending. Player lamps are the main light source.

Shadow budget: at most 6 shadow-casting lights visible at once (4 player
lamps, Brazier, one spare). All other lights have `Shadows = false`.

- [ ] **Step 1:** Ask the user to complete H-5. Confirm with `inspect_instance` on `Lighting`.
- [ ] **Step 2:** Apply the rig. In play mode capture six reference shots: Parlor by lamp, Cannery lane, LampHall with the Brazier lit, Yard in rain, Tower stair looking up, Cove.
- [ ] **Step 3:** Check each shot: black areas keep some shape within 20 studs of a lamp; nothing is pure white except flame cores; fog hides the far shore; no light leaks through walls (thicken or add a shadow-casting backing part where it does).
- [ ] **Step 4:** Check at graphics levels 3, 7 and 10 (ask the user to change the Studio setting): the game stays playable and fair at 3 (not darker than at 10).
- [ ] **Step 5:** Commit: `feat: add final lighting and post-processing`.

---

### Task E10: Gameplay prefab art

**Files:** Create `tools/mapgen/Prefabs/Items.luau`, `Brazier.luau`, `Cage.luau`, `DoorKit.luau`, and the art builds of `Keeper.luau`, `Brineling.luau`, `Wisp.luau`, `CarryProxy.luau` (models only; M12 does rigs and animation).

| Prefab | Look | Requirements |
|--------|------|--------------|
| `OilCanister` | Dented square tin, brass cap, faded label with a whale silhouette | Readable at 15 studs in lamp light |
| `VotiveCandle` | Thick tallow candle in a tin holder | `Flame` attachment with a light and particles, enabled by `Lit` |
| `FieldDressing` | Canvas roll with a plain green band | No red cross (protected emblem) |
| `LampFlask` | Small brass flask | |
| Keys | Iron keys with a shaped tag: fish, tower, barrel, boat | The tag shape identifies the key without text |
| `Brazier` | Iron tripod bowl 6 across, three sconce arms | `Bowl`, `Sconce1..3`, `Fire` attachment |
| `WispCage` | Rusted fish-trap cage of iron and net, 10 × 8 × 8 | `Gate` slides up |
| Doors | Panel doors (house), steel doors (Cannery), plank doors (Boathouse, Cellar), iron gate (Tower, `SeeThrough` bars) | Pivot exactly at `Hinge` |
| Storm Lamp | Brass hand lantern with a shutter lens | Used by the viewmodel (U9) and on other players' hands |
| Keeper, Brineling, Wisp | As plan 04, M12 | Within 20,000 triangles per mesh; Keeper ≤ 60,000 total |

Every item has a soft highlight when it is the interaction target (client,
U3) — the prefab only needs a clean silhouette.

- [ ] **Step 1:** Build each prefab into `ServerStorage.Prefabs`. Keep the greybox root parts, attachments and names so code does not change.
- [ ] **Step 2:** Run Q3 (prefab section). `screen_capture` each prefab on a turntable under lamp light.
- [ ] **Step 3:** Commit: `feat: add gameplay prefab art`.

---

### Task E11: Visual effects

**Files:** Create `src/client/Controllers/VfxController.luau`, `src/client/Vfx/*.luau`, effect templates in `ReplicatedStorage.Assets.Vfx` (built by `tools/mapgen/Prefabs/Vfx.luau`).

| Effect | Trigger | Description |
|--------|---------|-------------|
| Lamp light | `LampOn` on any player | `SpotLight` on that player's lamp (spec 6 values). Shadows on |
| Beam | `BeamOn` | Tighter, brighter spot; a visible cone (`Beam` with a soft gradient texture); small dust particles in the cone |
| Lamp low | `LampCharge` < 20 | Light flickers |
| Brazier fire | `BrazierLit`, `BrazierCharges` | Flame height grows with charges (0: embers, 3: tall) |
| Wisp | Wisp `State` | Soft blue-white glow, slow trail. `Flee`: flickers. `Follow`: leans to the candle. `Released`: bright rise and fade |
| Release wave | `Cue("WispRelease")` | A cold light pulse from the Brazier; all `FlickerLight`s dim for 2 s |
| Keeper stage cues | Keeper `Stage`, `Ember`, `State` | As plan 04 M12. Stun: steam off the body |
| Brineling | `State` | Wet sheen; `Burning`: white steam and shrink; no fire or gore |
| Latch | Local `LatchMeter` > 0 | Dark vignette and wet-edge overlay scaled by the meter |
| Chased | Local `Chased` | Slight vignette pulse; FOV unchanged |
| Rain | Outdoors (local `ZoneMath`) | Particles on a part above the camera; splashes on the ground; none indoors; window streaks indoors |
| Lightning | Deterministic from server time: `Random.new(math.floor(serverTime / 30))` gives the offset in each 30 s window | `Lightning` lights flash twice in 0.2 s; sends a local event that AudioController uses for thunder |
| Flicker | `FlickerLight` groups | Per-group noise on brightness |
| Door, item | Attribute changes | Dust puff on door close; glint on items in lamp light |

Rules: all effects are client-side and read replicated attributes. With the
photosensitivity setting on, lightning and flicker are replaced by slow
fades. Particle counts halve below graphics level 5. Effects are pooled; no
`Instance.new` per frame.

- [ ] **Step 1:** Implement each row with a T3 screenshot check.
- [ ] **Step 2:** Commit per group: `feat: add <group> effects`.

---

### Task E12: Optimisation and art review

- [ ] **Step 1:** Measure with the Game Tester's perf script (Q10) at the six reference viewpoints. Record instance count, triangles, draw calls and frame time in `docs/qa/perf.md`.
- [ ] **Step 2:** Fix until every budget in spec 16.3 is met. Use, in this order: `CastShadow = false` on small props; `RenderFidelity = Automatic` on meshes; `CollisionFidelity = Box` or `Hull` on props; merging static clutter into single meshes; fewer particles; large simple wall backings so occlusion culling works indoors.
- [ ] **Step 3:** Art review: compare every zone with the reference shots and the art bible. List mismatches in `docs/art/review.md` and fix them.
- [ ] **Step 4:** Check the manifest: every external asset in the place has a row; every row has "Scripts removed: yes". Verify by script: no `LuaSourceContainer` under `Workspace.Map`, `ServerStorage.Prefabs`, `ServerStorage.PropLibrary` or `ReplicatedStorage.Assets`.
- [ ] **Step 5:** Re-run Q3 and `NavCheck`. Ask the user to save the milestone snapshot (H-7).
- [ ] **Step 6:** Commit: `perf: optimise map and close art review`.
