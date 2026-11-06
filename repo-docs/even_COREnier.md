# even_COREnier — CORE v2 game server (archived)

**Status:** ARCHIVED (moved into `monorepo/server`). 261 commits, Feb 2025 → Aug 2025.
**Stack:** C++ (Makefile build), nlohmann JSON.

## What it is
The second-generation authoritative game server, written in C++ after the Rust `core` server was abandoned. It accepts TCP connections from two or more bots, validates their login, runs the tick loop, executes actions, generates the world, and writes replay JSON files.

## Contents
- `src/main.cpp`, `inc/` — entry point and headers; `src/{action,config,game,net,object,utils}` (same layout as `monorepo/server/src`).
- `src/action/` — that era's rules: `CreateAction` (fixed unit types), `MoveAction`, `AttackAction`, `BuildAction` (building walls), `TransferMoneyAction` ("money" instead of gems).
- `src/game/ReplayEncoder.cpp`, `StateEncoder.cpp` — diff-based replay/state format (referenced by the visualizers); `Visualizer.cpp`.
- `data/game.config.json`, `data/server.config.json` — configs of that era.
- `data/jigsaw-templates/*.txt` — ASCII map pieces for the jigsaw world generator (circle, arch, arrow, walled resources…).
- `misc/` — tooling.
- `entrypoint.sh`, `.github/workflows/` — Docker image for the server.

## Run
`make`, then `./server <server config path> <teamId1> <teamId2> ...` (the current monorepo server takes different arguments).

## Relevance to the competition: ★☆☆☆☆
Superseded by `monorepo/server`, which has newer mechanics (component-based units, custom names, updated cooldown rules). Only useful to read history of rule changes with `git log`.
