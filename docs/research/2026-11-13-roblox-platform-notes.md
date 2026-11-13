# Roblox platform notes (checked 2026-11-13)

Facts an implementing agent needs. "UNCONFIRMED" means it was not verified
against a source in the research session. **Rule for every task:** before
using an engine API, open its page on `create.roblox.com/docs/reference` and
confirm it. If the docs disagree with this file, the docs win; record the
difference in `docs/DECISIONS.md`.

## 1. File-based workflow

- **Rojo 7.7.1** is the latest release. 7.7.1 fixes Windows `\\?\` path bugs,
  so 7.7.1 is the minimum. 7.7.0 added `rojo syncback`.
  https://github.com/rojo-rbx/rojo/releases
- **Rokit** is the current toolchain manager (`rokit.toml`). Aftman is legacy.
  https://create.roblox.com/docs/projects/external-tools
- Wally, Selene, StyLua, luau-lsp, Lune and Asphalt are third-party tools.
  Their current version numbers were not checked: resolve them with
  `rokit add` and commit the pinned result.
- `rojo build -o x.rbxl` produces a binary place. Open Cloud accepts `.rbxl`.

## 2. Testing and agent control

- **TestEZ** was archived on 14 Sep 2024. Do not use it.
- **Jest-Lua**: `jsdotlua/jest@3.10.0` and `jsdotlua/jest-globals@3.10.0` on
  Wally. Runs only inside Roblox. Globals are imported from JestGlobals.
  https://github.com/jsdotlua/jest-lua
- **Open Cloud Luau Execution** runs a script headlessly against a place:
  - Create: `POST https://apis.roblox.com/cloud/v2/universes/{u}/places/{p}/luau-execution-session-tasks`
    with body `{script, timeout}`.
  - Poll: `GET .../versions/{v}/luau-execution-sessions/{s}/tasks/{t}`; logs at
    `/logs`. States: QUEUED, PROCESSING, COMPLETE, FAILED, CANCELLED.
  - Auth: header `x-api-key`, scope
    `universe.place.luau-execution-session:write`. A key without the scope may
    return success without acting, so always assert on the returned result.
  - Limits: 5 creates per minute, 5 minutes runtime, 4 MB script, 4 MB return,
    10 incomplete tasks per place, 450 KB logs.
  - **Physics does not run and scripts do not auto-run.** Use it for unit and
    logic tests only.
  - https://create.roblox.com/docs/cloud/reference/features/luau-execution
- **Studio MCP server** is built into Studio. https://create.roblox.com/docs/studio/mcp
  - Enable: Assistant > … > Manage MCP Servers > Enable Studio as MCP server.
  - Transport is stdio. On Windows: `cmd.exe /c %LOCALAPPDATA%\Roblox\mcp.bat`.
  - Tools: `execute_luau` (Edit, Client or Server context), `start_stop_play`,
    `get_studio_state`, `get_console_output`, `screen_capture`,
    `character_navigation`, `user_keyboard_input`, `user_mouse_input`,
    `script_read`, `multi_edit`, `script_grep`, `script_search`,
    `search_game_tree`, `inspect_instance`, `search_asset`, `insert_asset`,
    `upload_image`, `generate_mesh`, `generate_material`, `generate_animation`,
    `subagent`, `save_place`, `list_roblox_studios`.
  - Every call needs a `studio_id`. Teleport testing is limited.
- run-in-roblox: status UNCONFIRMED. Not used.

## 3. Lighting and graphics

- `Lighting.Technology` is deprecated. Use `LightingStyle` (`Realistic` or
  `Soft`) and `PrioritizeLightingQuality`. **Scripts cannot set these.** Set
  them in Studio's Properties panel.
  https://create.roblox.com/docs/reference/engine/classes/Lighting
- Dark-interior properties: `Ambient`, `OutdoorAmbient`, `Brightness`,
  `EnvironmentDiffuseScale`, `EnvironmentSpecularScale` (metalness maps do
  nothing at 0), `ExposureCompensation` (−5 to 5), `GlobalShadows`,
  `ShadowSoftness`. Light range is clamped to 120 studs.
- **SurfaceAppearance**: Color, Normal (OpenGL tangent space only), Metalness,
  Roughness, emissive mask. At runtime only `Color`, `AlphaMode`,
  `EmissiveStrength`, `EmissiveTint` and `ResampleMode` are scriptable.
- **Occlusion culling** is live for parts and meshes. It works best indoors
  with large, simple occluders. One report links it to light flicker.
- UNCONFIRMED: official volumetric lighting or global illumination (none
  found), MaterialVariant details, the full post-effect list,
  EditableMesh/EditableImage status.

## 4. Networking and performance

- Streaming is on by default for new places. Bleak turns it **off**
  (see master plan, decision D-07).
- Remotes: about 500 `FireServer` requests per second per client, shared across
  all RemoteEvents. UnreliableRemoteEvent has a separate budget and a
  documented 900-byte payload cap.
- The engine "Server Authority" feature (prediction and rollback) is reported
  released in 2026 with rough edges. Bleak does **not** use it (decision D-08).
- Parallel Luau exists (Actors). Not needed at Bleak's scale.
- UNCONFIRMED: `BasePart:SetNetworkOwner(nil)` for NPCs. Long-standing API;
  confirm on the docs page before use.

## 5. NPC AI

- PathfindingService agent parameters: `AgentRadius`, `AgentHeight`,
  `AgentCanJump`, `AgentCanClimb`, `WaypointSpacing`, `Costs`
  (`math.huge` blocks a label).
- `PathfindingModifier` with a `Label`, and `PassThrough` for doors.
  `PathfindingLink` joins two attachments.
- Pitfalls: 3,000-stud straight-line cap, 20,000-node budget. Recompute on
  `Path.Blocked` only when the blocked index is at or beyond the next waypoint.
  https://create.roblox.com/docs/characters/pathfinding
- UNCONFIRMED: shapecast APIs, Animator usage on custom rigs, production
  status of ControllerManager. Bleak uses the classic Humanoid.

## 6. Audio

- The Audio API (AudioPlayer, AudioEmitter, AudioListener, AudioDeviceOutput,
  joined by Wire) is out of beta.
- **Acoustic Simulation** fully released 22 Sep 2026. Enable with
  `SoundService.AcousticSimulationEnabled = true`. It gives occlusion,
  diffraction and reverb for Audio API instances only. Tuning:
  `BasePart.AudioCanCollide`, `PhysicalProperties.AcousticAbsorption`.
  Quality drops or switches off at low graphics levels, so **gameplay must not
  depend on it**. https://devforum.roblox.com/t/4307121
- Uploads: mp3, ogg, wav or flac, up to 7 minutes, 100 per month when
  ID-verified (10 total otherwise), private until moderated, permission granted
  per experience. https://create.roblox.com/docs/audio/assets
- Voice chat needs the user age check and is enabled in Experience Settings.

## 7. Input and UI

- The Input Action System is reported fully released on 29 Jun 2026 (thread
  title only verified). Bleak wraps all input in one module so the backend can
  be changed in one place (decision D-09).
- UNCONFIRMED this session but long-standing: ContextActionService,
  UserInputService, ProximityPrompt, `Enum.CameraMode.LockFirstPerson`.

## 8. Data and publishing

- DataStore limits: per server 60 + players × 40 requests per minute.
  4,194,304 characters per key.
- Place publish by Open Cloud:
  `PATCH https://apis.roblox.com/assets/v1/assets/{placeId}`, multipart.
  **It does not update SurfaceAppearance, EditableImage/Mesh, PartOperation or
  BaseWrap.** A release build must be published from Studio.
  https://create.roblox.com/docs/cloud/guides/usage-place-publishing
- Content maturity: jump scares, creepy NPCs and ominous music are Mild fear.
  Any realistic blood is at least Moderate. Heavy realistic blood is
  Restricted (18+). The questionnaire is mandatory.
  https://create.roblox.com/docs/production/promotion/content-maturity

## 9. Asset pipeline

- Assets API: `POST https://apis.roblox.com/assets/v1/assets`, then poll
  `/operations/{id}`. 20 MB per asset. Models as fbx, gltf, glb or rbxm.
- Asphalt (`asphalt.toml`, `ASPHALT_API_KEY`) uploads images, audio,
  animations and models and generates Luau id tables.
  https://github.com/jackTabsCode/asphalt
- Meshes: 20,000 triangles each, one material per mesh.
- Textures: docs contradict themselves (4096 vs 1024). Budget **1024 × 1024**.
  Maps: 24-bit albedo, 24-bit OpenGL normal, 8-bit roughness, metalness and
  emissive mask.
