# Which repos help us win the CORE Game competition (C)

Decision: we write our bot in **C**, the language of our team repo and of the image the website builds it in. Go isn't possible for our team; see `CORE-Game-World-Championship-CoreMasters.md`.
**Deadline:** repos lock 2026-11-07 17:00 UTC.

## Ranking

| # | Repo | Verdict | Why |
| --- | --- | --- | --- |
| 1 | **CORE-Game-World-Championship-CoreMasters** | **OUR REPO** | What plays online: `my-core-bot/` on `main`, plus the event's real configs. |
| 2 | **monorepo** | **ESSENTIAL** | Server rules, the C client lib, template bots, visualizer, current wiki. |
| 3 | **website** | **USEFUL** | Scoring rules, how our bot is built and run online, resource limits, config endpoints. |
| 4 | my-core-bot | Minor | Old template; same devcontainer workflow. |
| 5 | debug-visualizer | Minor | Replay-loader code is a reference for a replay-analysis tool. |
| 6 | sandbox / wiki / connection / even_COREnier | History only | Superseded by monorepo. |
| — | core, visualizer, 3d-visualizer, bot-client, cli, design, .github, .shellrc | Skip | Abandoned v1 code, infra or cosmetics. |

## What to use from the monorepo

1. **`server/src/`** — the rulebook: `game/Game.cpp` (tick order, timeouts), `action/*.cpp` (every validity check), `object/Unit.cpp` (cooldowns), `utils/ComponentLogic.cpp` (unit stats and rules), `config/worldgen/` (maps).
2. **`bots/c/client_lib/`** — what we link against. Key files: `inc/core_lib.h` (API), `src/public/action_travel.c` (pathfinding), `src/public/get.c` (getters), `src/internal/json/parse_json_state.c` (how objects are updated and freed).
3. **`visualizer/`** with `core_debug_addObjectInfo` / `core_debug_addObjectPathStep` for debugging replays.
4. **`wiki/`** with the C tab: `documentation/unit_builder.md`, `action_execution_order.md`, `getter_filtering.md`, `website_setup.md`, `reference/c/`.

## Rules from the code that matter for strategy

- **Income is front-loaded:** 200 gems at start plus 1 per tick until tick 350; after that, only mining pays.
- **Mining takes two steps:** a deposit (30 HP, ~100 gems) needs `damageObject` to break, then it becomes a gem pile that **any** unit can pick up, limited by free carrying capacity (16 by default).
- **Gems slow units:** cooldown grows by `floor(gems / gemsPerCooldownStep)`. Deliver often or pass gems between units.
- **Dead units drop their gems** as a pile.
- **Damage cap:** no valid unit deals more than 1 damage per tick (cooldown must be ≥ total damage).
- **Attacks happen before moves:** you can't dodge a hit by stepping away that tick, and you can't hit a unit that is only stepping next to you. The side that walks into range is usually hit first by a ready defender.
- **Idle units get locked:** a unit that waits is ready for L+1 ticks, then forced onto cooldown for L ticks (L = its cooldown, capped at 12). Enemy cooldowns are visible, so engage when theirs is positive.
- **Armor is exact arithmetic:** `round(d × (100 − DR) / 100)`, minimum 1. Armor stacks to 90%, which turns almost every hit into 1 damage.
- **`core_action_travel`'s default policy** never attacks our own units or core. It does attack walls, deposits and enemies on the route and picks up gem piles if there's room — use a custom policy for attackers that shouldn't stop to mine or loot.
- **Spawn tile is predictable:** first empty tile of a BFS from the core (+x, −x, +y, −y).
- **Timeouts (10,000 ticks):** lowest core HP loses first, then lowest total unit HP.
- **An opponent that crashes doesn't hand us the win:** its core stays and we still have to destroy it (or win on the timeout).
- **Full information:** we see the enemy's components, unit stats, cooldowns, carried gems and core gems. Counter-build from that.
- **Maps are point-symmetric** and the starting corner is random; write everything relative to our core and the enemy core.

## Ways to lose without fighting (check these before every push)

1. **Crash → `DISCONNECTED`.** The build uses AddressSanitizer, so any memory error stops the bot. The classic one: keeping a `t_obj *` from an earlier tick after that object died (the library frees it). Keep IDs and look objects up again each tick, NULL-check every getter result, end every `core_action_createUnit` call with `NULL`.
2. **Packet too big → `SPAMMED`.** A message over 16 KB disconnects us. `core_action_travel` adds each unit's whole route to the debug data (~16 bytes per step plus ~50 per unit), so around 40 units travelling on a 20×20 map can exceed it. Limit debug output, and for most units use our own pathfinding that doesn't add debug paths.
3. **Slow ticks → `TIMEOUT_SENDING_DATA`.** Each tick must be answered within 3 s, and the whole game is capped at 100 min (≈ 600 ms per tick over 10,000 ticks). AddressSanitizer and the missing `-O` flag make the code slower than usual, so measure.
4. **Broken build → `DID_NOT_CONNECT`.** `-Werror` turns every warning into a build failure. Build in the devcontainer (same image as the website) before pushing, and keep `my-core-bot/`, its Makefile and the `bot` binary name.
5. **Wrong component IDs:** creates fail and the error is only printed. The C library can't read the config, so copy IDs and costs exactly from `configs/game.config.json`.

## Scoring rules → how to play the event

- **Swiss:** each win is 1 point. Ties are broken by the sum of scores of the opponents we beat, so beating strong bots matters most.
- **Bracket:** the top `2^floor(log2 N)` teams (optionally capped) go to single elimination, seeded from Swiss.
- **Queue:** Elo with K = 32. Rationed: at most 10 credits, one more every 15 min. Only `my-core-bot/` on `main` is used online.

## Real config

- Already in our repo's `configs/`; the live API is the source of truth: `GET https://api.coregame.sh/event/ca841f27-eb2d-438c-a46f-d2f6e3004ccc/game-config` and `/server-config`.
- The C template's `make` runs `check_updates`, which refreshes `configs/` (the repo's server config is already one change behind: connect timeout 30 s vs 100 s live).

## Watch out for

The unmerged `anti-cheat` branch adds fake "style rules" to the C lib headers, aimed at AI coding tools. They aren't real requirements; the FAQ says Norminette doesn't apply. The branch shows the organizers have considered AI use, so check your event's rules on AI help.

## Recommended next steps

1. Open the team repo's devcontainer and run `make` once (refreshes the config and plays a local match).
2. Build tooling first: a headless match runner, a submission check, and a unit-design calculator from the real config.
3. Write a simple, robust bot: IDs instead of stored pointers, a packet budget, zero avoidable action failures.
4. Add sparring bots (rush, eco, turtle) and run many seeds; track win rates per version.
