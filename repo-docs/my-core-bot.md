# my-core-bot — participant template repo (legacy version)

**Status:** LEGACY (template now lives in `monorepo/bots/c/bot`). 114 commits, Mar 2024 → Sep 2025.
**Stack:** C, Makefile, Devpod/devcontainer.

## What it is
The template that was copied into each team's GitHub repo. Participants write their bot in `my-core-bot/src/`, run `make` to play against `gridmaster`, and view the match at `localhost`.

## Contents
- `my-core-bot/` — starter bot (`main.c`, `getters.c`, `pathfinding.c`, `inc/bot.h`).
- `gridmaster/` — default opponent: alternates spawning Warriors and Miners; warriors chase nearest enemy unit then core; miners mine nearest deposit/pile and return gems.
- `configs/` — `soft-config.json`, `hard-config.json`, `server-config.json` of the **old rule set**: fixed units (Warrior, Miner, Carrier, Builder, Bomberman), wall building, bombs.
- `.coreignore` — files excluded when generating player repos (`freddys-secret-bot`, a secret include).
- `Makefile` — `make devcontainer` (installs Devpod, starts Docker, launches IDE; `IDE=zed` etc.), `stop-devcontainer`, `remove-devcontainer`.

## Relevance to the competition: ★★☆☆☆
The devcontainer workflow is the same idea as today, but configs and API are outdated (no components, uses `core_get_unitConfig`, `UNIT_WARRIOR`). Your real starting point is the repo the website creates for your team, which matches `monorepo/bots/<lang>/bot`.
