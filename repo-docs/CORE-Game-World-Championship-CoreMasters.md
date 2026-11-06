# CORE-Game-World-Championship-CoreMasters — our team repo

**What it is:** the repository coregame.sh generated for our team **CoreMasters** in the event **CORE Game World Championship**.
Remote: `git@github.com:42-core-repos/CORE-Game-World-Championship-CoreMasters-4a8cc30a-9eeb-4c24-8ee0-20bb0baf4ca8.git`.
History: one commit ("Initial commit", 2026-11-06 17:00 UTC, by the organizer account `corethegame`), branch `main` only.
**Whatever is on `main` in `my-core-bot/` is what plays online.**

## Event facts (public API, checked 2026-11-06 19:05 UTC)

| Item | Value |
| --- | --- |
| Event | CORE Game World Championship, id `ca841f27-eb2d-438c-a46f-d2f6e3004ccc`, public |
| State | `PLAYING` since 2026-11-06 17:00 UTC |
| **Submission deadline / repo lock** | **2026-11-07 17:00 UTC** (24 h of coding) |
| Showtime / scheduled end | 2026-11-07 19:30 UTC |
| Team size | 1–2 |
| Versions | server, bot image and visualizer all `v0.1.3.0`, which is monorepo `dev` HEAD, so the rules in `monorepo.md` apply exactly |
| Event default template | `bots/c/bot`, image `ghcr.io/42core-team/my-core-bot-c:v0.1.3.0` |
| Queue | enabled; credits up to 10, one every 15 min |
| Betting ("gambling") | enabled |

## Contents

An exact copy of `monorepo/bots/c/bot` at `v0.1.3.0` (diffed). The only changes: team name `CoreMasters` in `my-core-bot/src/main.c`, event URLs in `scripts/`, image tags pinned to `v0.1.3.0`, and the event configs in `configs/`. No extra or hidden files. `.coreignore` is absent.

| Path | What it is |
| --- | --- |
| `my-core-bot/` | Our bot, in **C**: `src/main.c`, `src/getters.c`, `inc/bot.h`, `Makefile` → builds `bot`. |
| `gridmaster/` | Default local opponent (C). |
| `configs/` | `game.config.json`, `server.config.json` — the event's real configs. |
| `scripts/` | `check_update_configs.sh` and `check_image_updates.sh`, pointing at `https://api.coregame.sh/event/ca841f27-…`. |
| `Makefile` | `make` = fetch config updates, build both bots, run the server and both bots; `make devcontainer` etc. |
| `.devcontainer/` | `my-core-bot-c:v0.1.3.0` + `visualizer:v0.1.3.0` on `localhost:4000`. |
| `.vscode/` | Hides most files, including `configs`, `gridmaster`, `scripts` and `Makefile`. |
| `.clang-format`, `.gitignore`, `replays/` | Formatting, ignores (`bot`, `build/`, `replays/replay_latest.json`). |

## Language: C (decided 2026-11-06)

We write the bot in C. Why Go was not an option:

- The website picks the image our bot is built in online per team, when the team is created: the team's starter template, else the event default (`api/src/match/match.service.ts`). Ours is `my-core-bot-c:v0.1.3.0`.
- That image has gcc and the C library but **no Go toolchain** (`bot-base` + C lib only). A Go bot pushed here would fail `make` online → `DID_NOT_CONNECT` in every match.
- The API has no way to change a team's template after creation; only team creation takes `starterTemplateId`. The list of templates for the event requires login, so I couldn't check whether a Go template is offered.

## Real game config (`configs/game.config.json`)

- **Map:** 20×20, `jigsaw` generator, cores at (0,0) and (19,19). 36 deposits and 16 gem piles in total (mirrored, so 18 + 8 per side). Deposit: 30 HP, ~100 gems (±20; +6 per adjacent wall; ×1.3 if fully surrounded). Gem pile: 25 ±8 gems. Wall: 42 HP.
- **Economy:** 200 gems at start, plus 1 per tick until tick 350 → 550 gems from idle income in total.
- **Core:** 99 HP.
- **Units:** base cost 42; defaults HP 4, cooldown 4, carry 16 gems, +1 cooldown per 16 carried gems, core spawn cooldown 16. Up to 6 components. `maxUnitStandingCooldown` 12.

| Component | Effect | Cost |
| --- | --- | --- |
| `health` | +6 HP, +3 spawn cooldown | 40 |
| `speed` | −2 cooldown, +12 spawn cooldown | 55 |
| `armor` | +30% damage reduction, +2 cooldown, +4 spawn cooldown (max 3) | 35 |
| `combat` | +3 damage vs units, +3 cooldown, +4 spawn cooldown | 35 |
| `siege` | +3 damage vs cores, +3 cooldown, +4 spawn cooldown | 35 |
| `demolition` | +3 damage vs deposits/walls, +3 cooldown, +4 spawn cooldown | 35 |
| `gem_speed` | +111 gems per cooldown step, −2 HP, +2 spawn cooldown | 40 |
| `max_gems` | +150 carry capacity, −1 HP, +2 spawn cooldown | 40 |

Rules: cooldown ≥ total damage; HP ≥ 1; at most 3 armor.

### What it implies (computed from this config with a script)

- **No unit can deal more than 1 damage per tick**, because cooldown must be ≥ total damage. `combat` alone (77 gems) does 0.43 per tick; `combat` + 2×`speed` (187 gems) reaches 1.0, but the core then waits 44 ticks before the next spawn.
- **Armor caps at 90%,** which turns almost every hit into exactly 1 damage (the minimum). 3×`armor` + 3×`health` gives 22 HP that takes ~22 hits to kill, but that unit has cooldown 10 and no damage.
- **Carriers:** `max_gems` + `gem_speed` carries 166 gems with no slowdown up to 126, but has 1 HP (adding `health` → 7 HP, 162 gems).
- **Mining one deposit:** 70 ticks with `demolition` (77 gems), 30 ticks with `demolition` + 2×`speed` (187 gems).
- **Spawn cooldown is a real limit:** each `speed` adds 12 ticks.
- Template units: "Warrior" = `combat` (77 gems, 4 HP, cooldown 7); gridmaster's "Gridfighter" = `combat`+`siege`+`health` (152 gems, 10 HP, cooldown 10).

## Real server config (`configs/server.config.json`)

- `timeoutTicks` 10,000; `timeoutMs` 6,000,000 (100 min ≈ 600 ms per tick over a full game).
- `clientWaitTimeoutMs` **3,000** — every tick must be answered within 3 s.
- `clientConnectTimeoutMs`: 30,000 in the repo copy, **100,000 on the live API**. The repo copy is stale; `make` updates it through `check_updates`.
- `clientPacketsMaxSizeKb` 16; login timeout 5 s (server default).

## Relevance to the competition: ★★★★★
This is where our bot lives and what the tournament runs.
