# core — CORE v1 game server in Rust (abandoned)

**Status:** ABANDONED ("CORE v1 Game Server has been abandoned"). 424 commits, Oct 2023 → May 2025.
**Stack:** Rust, tokio (async TCP), Cargo; devcontainer.

## What it is
The original game server. Same concept (bots connect over a socket, server runs ticks and broadcasts state), but an older rule set: fixed unit types, resources, passive income, a `travel` action handled server-side, spectator connections.

## Contents
- `src/game/main.rs` — binary entry.
- `src/core/game/` — `game.rs` (loop), `bridge*.rs` (connections), `login.rs`, `config.rs`, `generate.rs` (map), `passive_income.rs`, `spectator.rs`, `state.rs`, `team.rs`.
- `src/core/game/action/` — `attack`, `create`, `travel`, `request`.
- `src/core/game/entity/` — `core`, `unit`, `resource`.
- `src/tests/` — tests; `wiki/` submodule.
- Env vars: `PORT`, `LOG_TO_FILE`, `LOG_LEVEL`, `SEED`, `TICK_RATE`.

## Run
`make build` / `make run` / `make test` (or open the devcontainer).

## Relevance to the competition: ☆☆☆☆☆
Different game rules from today. Historical only.
