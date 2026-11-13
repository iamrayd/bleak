# Genre reference: DEVOUR (Straight Back Games, 2021)

Researched 2026-11-13. Purpose: understand the mechanics of the co-op
"ritual under escalating threat" genre so Bleak can be an **original** game in
that genre.

## Source quality

- The DEVOUR fandom wiki returned HTTP 402 and several guide sites returned
  403, so they were **not** read.
- Facts below come from the Steam page, Wikipedia and guide sites, some through
  search snippets only.
- Anything marked **UNCERTAIN** is single-source, conflicting or unconfirmed.
- Second-by-second numbers (stun duration, UV drain, movement speeds) were not
  found in any reachable source. Bleak's numbers are therefore Bleak's own
  design, not copies.

## 1. Core loop

- 1–4 player online co-op PvE with a single-player mode. Sessions run up to
  about an hour.
- Objective: destroy 10 ritual items to exorcise a possessed cultist.
- Lose: all players downed at once. The Steam blurb says "before time runs
  out"; no hard timer was confirmed. UNCERTAIN.
- First map order of play:
  1. Find randomly placed keys to unlock rooms.
  2. Light the altar with fuel.
  3. Release the ritual animals from two holding places (5 + 5), one of which
     needs its own key.
  4. Lure each one to the altar with a lure consumable.
  5. Burn it, spending fuel.
- The antagonist and minions are dormant until players approach the antagonist
  or take a specific key (sources differ slightly).

## 2. Main antagonist

- Roams, then switches to chasing a player. A UV light staggers it.
- Each ritual item destroyed makes it faster, more aggressive and harder to
  stagger, and minions spawn more often. Its look changes as a visual cue.
- A rage fit follows each destruction.
- UV stuns needed to calm it rise with progress. Sources conflict on the
  thresholds (1–3 items = 1 stun, 4–7 = 2, 8+ = 3 versus 1–4 = 1, 5–6 = 2,
  7+ = about 3). UNCERTAIN.
- A late "red eyes" state makes it faster and briefly immune to stun.
- Destroying several items back to back skips intermediate rage stages. Teams
  use this late in a run.
- A caught player is dragged to another location and left downed. On some later
  maps the player is killed outright instead.
- Guides advise turning the light off, standing still and listening, which
  implies light and movement feed detection. Exact rules are UNCERTAIN.

## 3. Minions

- Spawn at random and more often after each destruction.
- Need several seconds of contact to down a player.
- Destroyed by UV. In the hardest mode there is no UV, so they cannot be killed.

## 4. Player mechanics

- One held item at a time.
- Sprint with stamina.
- Flashlight with a UV mode, which is the only defence. UV is a finite charge
  that recharges slowly and is topped up by battery pickups.
- Downed players stay visible to teammates and are revived with a medkit.
- No permadeath on the base maps; a full team down is a loss.
- Keys are randomised. Some rooms stay locked until progress is made.
- Hiding is positional (dark, still). No dedicated hide-spot mechanic confirmed.
- Positional voice chat. No source confirmed that the AI hears voice. UNCERTAIN.

## 5. Ritual item behaviour

- The ritual animals roam, can be steered, follow or stay near the lure, and
  scatter when scared. One can be left at the altar with a lure, which lets a
  solo player manage two.
- Locked doors, ritual objects and item spawns are randomised per session.
- Medkits respawn (one 2021 player comment). Other respawns UNCERTAIN.
- Players stockpile consumables near the altar before destroying.

## 6. Difficulty

- Normal, Hard, Nightmare.
- Nightmare removes the UV light: the antagonist cannot be stunned or calmed
  and minions cannot be killed.
- Exact Hard changes not found. UNCERTAIN.

## 7. Map variations

Later maps keep the formula and change one variable: a different ritual item,
a two-step "prepare then destroy" process, or death instead of dragging.

## 8. Meta progression

XP raises a rank, rank grants tokens, tokens buy perks and flashlight
cosmetics. About 16 perks with effects such as speed while chased, faster long
interactions, slower minions nearby, and faster UV recharge.

## 9. Why it feels tense

- Progress is the thing that makes the threat worse, and the team chooses when
  to push.
- The stun tool weakens as progress rises, which forces a shift from fighting
  to evading.
- A friend being dragged away creates a risky rescue.
- Chase music and a tight, looping layout.
- One stray minion can end a run.

## 10. What Bleak takes and what Bleak changes

Bleak uses the **genre mechanics** above. It does not use DEVOUR's name,
characters, story, maps, art, audio or level layouts.

| Genre mechanic | Bleak's own form |
|----------------|------------------|
| 10 ritual items destroyed at an altar | 10 Wisps (drowned souls) released at a Beacon Brazier. Souls are freed, nothing is sacrificed |
| Living ritual items that flee and need a lure | Wisps flee players and follow a lit Votive Candle |
| Fuel consumable per destruction | Whale-oil canisters. The brazier stores up to 3 charges |
| Possessed humanoid that escalates per item | The Keeper, a lighthouse keeper held by "the Undertow". 10 tuned stages |
| UV flashlight stun | Storm Lamp with a focused Beam mode and a charge meter |
| Minions destroyed by light | Brinelings, salt-and-kelp crawlers that rise from drains |
| Dragged away and downed | The Keeper carries a caught player to a drop point |
| Medkit revive | Field Dressing, 4 s hold |
| Random keys and locked rooms | Four keys, placement proven solvable by a test |
| Normal / Hard / Nightmare | Same three tiers, with Bleak's own modifiers |
| XP, rank, perks | Six original perks, two equipped |
| Farmhouse and later maps | One original map: Greyharbor Light, a lighthouse rock |

Bleak additions that the reference does not have (as far as sources show):

- Anti-softlock rules that are tested, not assumed (spec section 14).
- A carried player can be freed mid-carry by stunning the Keeper.
- Solo "Last Breath" self-revive charges.
- No blood or gore, to fit Roblox content maturity rules.

## Sources

- https://store.steampowered.com/app/1274570/DEVOUR/
- https://en.wikipedia.org/wiki/Devour_(video_game)
- https://gamerant.com/devour-how-to-beat-the-farmhouse/
- https://gameplay.tips/guides/10520-devour.html
- https://www.cheatbook.de/files/devour.htm
- https://steamcommunity.com/app/1274570/discussions/0/3104637814592276908
- https://gamepur.com/?p=285471 (snippet only)
- https://gamersdecide.com/articles/devour-best-perks (snippet only)
- https://3rd-strike.com/?p=124543 (snippet only)
- https://gamesradar.com/have-you-tried-watching-a-possessed-cult-leader-drag-your-friends-into-an-attic-while-you-sacrifice-goats-in-devour (headline and snippet only)
