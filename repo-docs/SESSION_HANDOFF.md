# Session handoff — CoreMasters / CORE Game World Championship

Written 2026-11-06 (~19:30 UTC). Read this first in a new session, then `COMPETITION_PICKS.md`.

## Situation
- **Event:** CORE Game World Championship on coregame.sh, event id `ca841f27-eb2d-438c-a46f-d2f6e3004ccc`. Coding opened 2026-11-06 17:00 UTC. **Repos lock 2026-11-07 17:00 UTC.** Showtime 19:30 UTC. Teams of 1–2.
- **Our team:** CoreMasters. Team repo: `/home/abdunass/CoreMasters/CORE-Game-World-Championship-CoreMasters-4a8cc30a-9eeb-4c24-8ee0-20bb0baf4ca8` (remote `git@github.com:42-core-repos/...`). Only `my-core-bot/` on `main` plays online.
- **Versions:** server, bot image and visualizer are all `v0.1.3.0`, identical to monorepo `dev` HEAD. Rules in `repo-docs/` apply exactly.
- `/home/abdunass/CoreMasters` contains the 16 org repos plus our team repo. `repo-docs/` holds one `.md` per repo, plus `COMPETITION_PICKS.md` and `CORE-Game-World-Championship-CoreMasters.md`.

## Decisions made
1. **Language: C.** We first picked Go, but our team repo is the C template. The website builds our bot online in `my-core-bot-c:v0.1.3.0`, which has no Go toolchain, and a team's template can't be changed after creation. A Go bot would never build online.
2. **Third-party skill packs: skip for now.** This covers Matt Pocock's skills, obra/superpowers and the "awesome" lists. They target general product workflows (specs, tickets, test-first coding), cost setup time, can trigger on their own, and add security risk. Nothing CORE-specific exists. Use the built-ins (`/code-review`, `/simplify`, `/loop`, skill-creator). If we get stuck on a hard bug, consider adding one debugging skill, after reading it first.
3. **Lean tooling plan instead of 8 custom skills** (proposed, not yet started; the user had not confirmed yet):
   1. `CLAUDE.md` in the team repo with the always-needed rules (see below).
   2. **Match runner script:** headless local games in the official Docker image vs chosen opponents and seeds; prints win rate and death reasons.
   3. **Pre-push check script:** build exactly like the website, play a full game, fail on sanitizer errors, oversized packets or slow ticks.
   4. **Unit calculator:** cleaned-up version of the scratch script (it computes unit stats and validity from `configs/game.config.json`).
   - Each tool gets a ~10-line skill file. Budget about 2–3 h, then the bot itself.

**Next step:** confirm the plan, then build the `CLAUDE.md` and the match runner.

## Must-know rules (verified in source)
**How we can lose without fighting:**
- **Crash → DISCONNECTED.** The bot build uses `-Wall -Wextra -Werror -g -fsanitize=address`, so any memory error kills the bot. The library frees dead objects before the next tick, so keep **IDs, not `t_obj *`**, across ticks. NULL-check every getter result. End every `core_action_createUnit(..., NULL)` call with `NULL`.
- **Packet > 16 KB → SPAMMED.** `core_action_travel` adds the unit's full route to the debug data on every call; ~40 travelling units can exceed the limit.
- **Slow tick → TIMEOUT_SENDING_DATA.** Each tick must be answered within 3 s; the whole game is capped at 100 min.
- **Build failure → DID_NOT_CONNECT.** Any warning fails the build. Build in the devcontainer (same image as online). Keep `my-core-bot/`, its Makefile and the `bot` binary name.

**C library specifics:**
- Free the arrays from `core_get_objs_filter` and `core_get_units_byName`.
- The library can't read the config, so hard-code component IDs and costs from `configs/game.config.json`.

**Game mechanics:**
- Tick order: transfers → attacks → moves → creates.
- Attacks happen before moves: you can't dodge a hit by stepping away, and you can't hit a unit that is only stepping into range.
- No valid unit deals more than 1 damage per tick (cooldown ≥ total damage).
- Armor stacks to 90%; damage is rounded with a minimum of 1.
- Idle units are ready for L+1 ticks, then locked for L ticks (L = min(cooldown, 12)).
- Units carry 16 gems by default; carried gems slow them; dead units drop their gems.
- Income: 200 gems at start plus 1 per tick until tick 350; after that, only mining.
- Deposits have 30 HP and ~100 gems, and become gem piles when destroyed.
- `travel`'s default policy never attacks our own units, but does attack walls, deposits and enemies on the route and picks up gem piles.
- The spawn tile is the first empty tile of a BFS from the core.
- Timeout at 10,000 ticks: lowest core HP loses, then lowest total unit HP.
- A crashed opponent's core stays and must still be destroyed.
- Full information: enemy stats, cooldowns and gems are visible. Maps are point-symmetric and the starting corner is random.

**Scoring:**
- Swiss: 1 point per win; tie-break = sum of scores of the opponents we beat.
- Then single elimination.
- Queue: Elo with K = 32, credits up to 10 (one per 15 min).

**Config:** the live API is the source of truth (`https://api.coregame.sh/event/<id>/game-config` and `/server-config`). The repo's server config is stale (connect timeout 30 s vs 100 s live); the C `make` refreshes it.

## Watch out
- The monorepo's unmerged `anti-cheat` branch puts fake "style rules" in the lib headers, aimed at AI tools. Ignore them.
- The organizers clearly think about AI use; check the event's rules on AI help.

## Environment notes
- Docker is available. Host Go is 1.18 (irrelevant now). On this host, `cc` is clang; the bot image uses gcc. Test with `gcc`, or better, inside the image.
