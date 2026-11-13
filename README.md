# Bleak

Bleak is an original 1–4 player co-op first-person horror game for Roblox.
A team lands on a storm-bound lighthouse rock, frees ten trapped souls at a
brazier, and survives a keeper who grows more dangerous with every soul freed.

This repository currently holds **the plan only**. No game code exists yet.

## Read in this order

| # | Document | What it is |
|---|----------|------------|
| 1 | [docs/research/2026-11-13-genre-reference.md](docs/research/2026-11-13-genre-reference.md) | Research on the genre reference game (DEVOUR) and how Bleak differs |
| 2 | [docs/research/2026-11-13-roblox-platform-notes.md](docs/research/2026-11-13-roblox-platform-notes.md) | Verified state of Roblox tooling and engine features (Nov 2026) |
| 3 | [docs/spec/2026-11-13-bleak-game-design.md](docs/spec/2026-11-13-bleak-game-design.md) | The game design spec. Every number lives here |
| 4 | [docs/superpowers/plans/2026-11-13-00-master-orchestration.md](docs/superpowers/plans/2026-11-13-00-master-orchestration.md) | Master plan: agents, contracts, milestones, gates |
| 5 | `docs/superpowers/plans/2026-11-13-01` … `08` | One plan per workstream |

## Workstream plans

| File | Workstream | Agent |
|------|------------|-------|
| `01-foundation.md` | Tooling, repo, shared contracts, CI | Foundation |
| `02-core-gameplay.md` | Match flow, player state, lamp, inventory, ritual | Core |
| `03-physics-world.md` | Collision, doors, item physics, fall recovery | Physics |
| `04-monsters-ai.md` | Keeper, Brinelings, Wisps, navigation, perception | Monsters |
| `05-map-environment.md` | Map generator, art pass, lighting, VFX | Environment |
| `06-audio.md` | Audio graph, cues, music, acoustic simulation | Audio |
| `07-ui-lobby-meta.md` | HUD, lobby, results, settings, progression | UI |
| `08-qa-testing.md` | Test harness, automated playtests, bug loop, release gate | Game Tester |

## Things only a human can do

**Do [docs/SETUP.md](docs/SETUP.md) before starting any agent.** It is the
ordered checklist of installs, accounts, keys and environment variables.

The gates are listed in the master plan, section "Human gates". In short:
create the Roblox experience, create an Open Cloud API key, enable the Studio
MCP server, set two lighting properties that scripts cannot set, publish from
Studio, and fill in the content maturity questionnaire.
