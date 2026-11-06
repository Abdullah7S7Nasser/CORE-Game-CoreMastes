# sandbox — submodule-based dev environment (archived)

**Status:** ARCHIVED (replaced by `monorepo`). 229 commits, Jan 2024 → Aug 2025.

## What it is
A dev workspace that pulled the server, client lib and visualizers in as **git submodules** so developers could run a full match locally.

## Contents
- `.gitmodules`: `core` → `even_COREnier`, `bot/connection` → `connection`, `3d-visualizer`, `debug-visualizer` (submodules are not checked out here).
- `bot/gridmaster/` — sample opponent with its own `pathfinding.c` (274 lines, hand-written path search).
- `bot/martin/` — empty starter bot (just `core_startGame`).
- `config.game.json`, `config.server.json`, `replays/`, `Makefile`, devcontainer.
- Workflow building the `my-core-bot` image.

## Relevance to the competition: ★☆☆☆☆
Fully superseded by the monorepo. The old `gridmaster/pathfinding.c` is a readable example of grid pathfinding, but the current lib's `core_action_travel` already does weighted Dijkstra.
