# connection — C client library (archived)

**Status:** ARCHIVED (now `monorepo/bots/c/client_lib`). 280 commits, Jan 2024 → Aug 2025.
**Stack:** C, Makefile, Doxygen.

## What it is
The C library participants linked into their bots. Handles the TCP socket to the server, login packet, JSON parsing of config/state, encoding actions, and exposes the public API (`core_lib.h`).

## Contents
- `inc/core_lib.h` (public API), `inc/core_lib_internal.h`.
- `src/con_lib.c` — startup/game loop.
- `src/internal/socket.c`, `utils.c`, `json/` (hand-written JSON encode/parse for login, actions, config, state).
- `src/public/actions.c`, `get.c`, `printing.c` — actions, getters, debug printing.
- `docs/` — scripts to generate Markdown docs from the header (`generate_md_docs.py`), `Doxyfile`.
- Workflow builds a Docker image with the compiled lib.

## Difference from current version
The archived API belongs to the old rule set: `core_action_createUnit(t_unit_type)` (fixed unit types), `core_action_attack(attacker, t_pos)`, `core_action_transferMoney`, `core_action_build`. No `core_action_travel` pathfinder, no debug path functions, no components. The monorepo version is the one bots are compiled against today.

## Relevance to the competition: ★☆☆☆☆
Use `monorepo/bots/c/client_lib` instead.
