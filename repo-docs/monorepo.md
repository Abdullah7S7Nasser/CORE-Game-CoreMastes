# monorepo — CORE Game (current, active)

**Status:** ACTIVE — the single source of truth for gameplay. 1,222 commits, last commit 2026-11-06, default branch `dev`, latest tag `v0.1.3.0`.
**Replaces:** `even_COREnier` (server), `connection` (C client lib), `debug-visualizer` (visualizer), `wiki`, `sandbox`, `my-core-bot` (template).

## What it is

Everything needed to *play* CORE in one repo: the C++ game server, the client libraries participants link against (C, Go, partial TS), the template bot repos that get copied into every team's repository, the 2D replay visualizer, and the wiki deployed at `wiki.coregame.sh`.

## Layout

| Path | What it is |
| --- | --- |
| `server/` | C++ game server (ex-`even_COREnier`). ~3.9k lines, CMake (fetches json-schema-validator on first build). |
| `server/data/jigsaw-templates/` | 100+ ASCII map pieces used by the `jigsaw` world generator. |
| `server/data/json-schemas/` | JSON schemas for configs and client packets (exactly what the server accepts). |
| `server/misc/jigsaw-builder/` | HTML tool for editing jigsaw templates. |
| `bots/c/client_lib/`, `bots/c/bot/` | C library (`core_lib.h`) and C template repo. |
| `bots/go/client_lib/` | Go client library, module `github.com/42core-team/go-client-lib`, package `core`, with tests. Active (Nov 2026: C/Go parity work). |
| `bots/go/bot/` | Go template repo: `my-core-bot/` (our bot), `gridmaster/` (default opponent), Makefile, devcontainer, scripts. |
| `bots/ts/` | TypeScript client — **stale** (still uses the old fixed `UnitType` enum WARRIOR/MINER/CARRIER/TANK). Don't use. |
| `bots/base/` | Files synced into every language template (`make -C bots sync-base`). |
| `bots/configs/fake_demo/` | Demo `game.config.json` + `server.config.json`. |
| `bots/configs/real/` | Organizer submodule (`../config.git`), not checked out here. Participants get the real config from the website API instead (see "Real config"). |
| `visualizer/` | Vanilla TS + Vite replay viewer (ex-`debug-visualizer`), Docker/nginx/Helm. |
| `wiki/` | Markdown docs with C/Go language tabs. |
| `.github/workflows/` | Builds the server image, bot base images, per-language template images and the visualizer image. |

## How to run locally

```bash
# inside the monorepo devcontainer (Ubuntu + Go 1.26 + server toolchain)
make LANGUAGE=c STAGE=dev     # builds the client lib, both bots and the server, then runs a match
make vis                      # visualizer (vite dev server)
```
- Player 1 = `bots/<lang>/bot/my-core-bot/bot` (id 42), Player 2 = `gridmaster` (id 418).
- Config: `bots/configs/real` if present, else `bots/configs/fake_demo`.
- **The template bots don't work with `fake_demo`:** they create units from `combat`/`siege`, which don't exist in that config. Every create fails, no units spawn, and the game idles until the 30,000-tick timeout. Use the real event config or change the component IDs.
- Go only: the host machine here has Go 1.18, which can't build the Go client lib (needs Go 1.26).

## Game rules — as implemented in the server

### Objects and map
- `OBJ_CORE`, `OBJ_UNIT`, `OBJ_DEPOSIT`, `OBJ_WALL`, `OBJ_GEM_PILE`. One object per tile, square grid (demo 20×20).
- Core positions come from the config's `corePositions` (demo: opposite corners) and are **assigned to teams at random**.
- The `sparse` and `jigsaw` generators fill the top-left triangle and mirror it through the centre: `(x,y) → (N−1−x, N−1−y)`. Sparse also keeps objects off the border, outside `coreBuffer` (Manhattan) of each core, and never next to another object (8-neighbourhood). Other generators: `hardcoded`, `empty`.
- **Full information:** both bots receive every object, including enemy units' components, properties, cooldowns and carried gems, and the enemy core's gems.

### Economy
- Core starts with `initialGems` (demo 300) and gains `idleIncome` (demo 3) per tick for the first `idleIncomeTimeOut` ticks (demo 100). After that, income comes only from mining.
- **Deposit:** has `depositHp`; hit it with `damageObject`. When destroyed it becomes a **gem pile** with `depositIncome` gems.
- **Gem pile:** any unit can "attack" it to pick up `min(pile, maxGems − carried)` gems; this uses the unit's action. The pile disappears when empty.
- **A dying unit drops all carried gems as a gem pile** on its tile.
- `transferGems`: from a unit or a core, to a core, a unit, a gem pile, or an empty tile (dropping; units only). Range: at most the distance from the target to *its* nearest empty tile — normally adjacent, further only if the target is boxed in. A unit transfer resets the unit's cooldown; core transfers have no cooldown.
- Carrying gems slows units (see Cooldowns).

### Units = components
- `CreateUnit(name, components...)`. Cost = `unitDefaultCost` + sum of component costs. At most `maxComponentsPerUnit` (demo 5). Failed creates cost nothing.
- Properties = defaults + Σ(component modification × count). The effective cooldown is never below 1, so speed components that push the base below 1 are wasted.
- `invalidConditions` are expressions over properties and component counts: `const`, `property`, `component_count`, `not`, `ternary`, `sum`, `subtract`, `multiply`, `divide`, `min`, `max`, `less_than`, `greater_than`, `less_than_or_equal`, `greater_than_or_equal`, `equal`. A create that matches one fails with that rule's message.
- Demo defaults: hp 5, cooldown 3, gemsPerCooldownStep 10, maxGems 100, all damage 0, postSpawnCoreCooldown 10. Demo components: `health` (+3 hp, 12), `speed` (−2 cooldown, 12), `armor` (+20% DR, 30), `unit_attack` (+3, 15), `core_attack` (+3, 15), `mining` (+3 object dmg, 15), `gem_carrying_efficiency` (+10 step, 20), `gem_carrying_capacity` (+50 max, 20). Demo rules: cooldown ≥ total damage; ≤ 4 armor.

### Tick order (`Game::tick`)
1. Debug info applied.
2. Actions executed in phases: **transfers → attacks → moves → creates**; random order within a phase (seeded).
3. Objects tick (idle income).
4. Dead objects removed (units → gem pile).
5. Timeout check.
6. Cooldowns advance.
7. State sent to clients, including that team's action-failure messages.

### Cooldowns (`object/Unit.cpp`)
- Acting sets the cooldown to `C = max(1, max(0, baseActionCooldown) + floor(gems / max(1, gemsPerCooldownStep)))`. It drops by 1 per tick and the unit can act at ≤ 0, so a unit acts at most once every C ticks.
- An idle unit keeps counting below 0, but on reaching −(L+1), where `L = min(C, maxUnitStandingCooldown)`, it is reset to +L. **So an idle unit is ready for L+1 ticks, then locked for L ticks.** Waiting gives no speed bonus; it only keeps the unit ready for a while.
- Enemy cooldowns are visible in the state.

### Combat (`action/AttackAction.cpp`)
- Attacks target an object **by ID**; range (Manhattan distance 1, no diagonals) and cooldown are checked when the action executes.
- Because attacks execute before moves: **a unit can't dodge a hit by stepping away that tick**, and **you can't hit a unit that is only stepping next to you this tick**. The unit that walks into range spends its action on the move, so a ready defender usually lands the first hit next tick.
- Damage by target type: unit → `damageUnit`, core → `damageCore`, deposit/wall → `damageObject`; a gem pile is a pickup.
- Armor: `round(d × (100 − DR) / 100)`, minimum 1 when d > 0 (a unit with 0 damage does 0). Rounding and the minimum make armor's effect depend on the exact damage: with 20% DR, hits of 1 or 2 lose nothing, while a 3-damage hit drops to 2. Compute it per matchup.
- Friendly fire: a direct `Attack` can hit our own units and our own core. `Travel`'s default policy never targets friendly units or cores.

### Spawning
- A new unit appears on the first empty tile of a BFS from the core, neighbour order +x, −x, +y, −y. The core then waits that unit's `postSpawnCoreCooldown` ticks.

### Winning, timeouts and disconnects
- The game ends when at most one core has HP > 0.
- After `timeoutTicks` (demo 30,000) or `timeoutMs` of wall-clock time (demo 50 min), the server kills one core per tick: **lowest core HP → lowest total unit HP → random.**
- Losing without a fight (replay `death_reason`):
  - `DID_NOT_CONNECT` — not connected within `clientConnectTimeoutMs` of server start (demo 30 s), or no login within `clientLoginTimeoutMs` (demo 5 s). Teams that never connect get no core.
  - `TIMEOUT_SENDING_DATA` — no packet within `clientWaitTimeoutMs` in a tick (demo 30 s).
  - `SPAMMED` — a single message larger than `clientPacketsMaxSizeKb` (demo 16 KB).
  - `DISCONNECTED` — socket closed (crash, panic, exit).
- **A disconnected team's core and units stay on the board.** The game continues until that core is destroyed or the timeout decides, so a bot must be able to finish off a passive core.

### Replays (`game/ReplayEncoder.cpp`)
- Written as `replay_latest.json` into the configured `replayFolderPaths`.
- Top-level keys: `misc` (`team_results` with `id`, `name`, `place`, `death_reason`; `stats`; `version`; `game_id`; `worldGeneratorSeed`), `ticks` (per-tick object diffs plus the executed actions), `config`, `full_tick_amount`.
- `death_reason`: 0 NONE_SURVIVED (winner), 1 CORE_DESTROYED, 2 DISCONNECTED, 3 SPAMMED, 4 DID_NOT_CONNECT, 5 TIMEOUT_SENDING_DATA, 6 TIMEOUT_CORE_HP, 7 TIMEOUT_UNIT_HP, 8 TIMEOUT_RANDOM.

## C client library (`bots/c/client_lib`) — what our bot uses

### API (`core_lib.h`)
- **Start:** `core_startGame(team_name, argc, argv, tick_callback, debug)`. Team ID from `argv[1]`, server from `SERVER_IP`/`SERVER_PORT` (default `127.0.0.1:4444`). Runs until our core dies or the server closes the connection.
- **State:** global `game` with `elapsed_ticks`, `my_team_id`, `grid_size`, `objects` (NULL-terminated `t_obj **`).
- **Objects:** `t_obj` has `type`, `data`, `id`, `pos`, `hp` and a union: `s_core` (`team_id`, `gems`, `spawn_cooldown`), `s_unit` (`team_id`, `gems`, `action_cooldown`, `components`, `properties`, `name`), `s_deposit_gems_pile` (`gems`). Walls have no extra data. Check `type` before reading the union.
- **Actions:** `core_action_createUnit(name, comp, ..., NULL)`, `core_action_move(unit, pos)`, `core_action_attack(attacker, target)`, `core_action_transferGems(source, pos, amount)`, `core_action_travel(unit, pos, policy)`. Nothing is checked locally; failures are printed in red on the next tick.
- **Getters:** `core_get_obj_from_id` (linear scan), `core_get_obj_from_pos` (O(1) index rebuilt every tick), `core_get_objs_filter`, `core_get_objs_filter_count` (NULL condition counts everything), `core_get_obj_filter_nearest` (Manhattan), `core_get_units_byName`, `core_get_units_byName_count`.
- **Debug:** `core_debug_addObjectInfo(obj, fmt, ...)`, `core_debug_addObjectPathStep(unit, pos)`.
- **No config access:** the library reads only `gridSize` from the config. Component IDs, costs and stats have to be taken from `configs/game.config.json` and hard-coded.

### Memory rules
- Free the arrays returned by `core_get_objs_filter` and `core_get_units_byName`, but not the objects in them. Don't free anything else the library returns, and never free `game` or its objects.
- Objects are updated in place, so a `t_obj *` stays valid while the object lives and its `data` field keeps its value between ticks.
- **When an object dies, the library frees it before our next tick.** A pointer kept from an earlier tick then points to freed memory, and with AddressSanitizer on, touching it crashes the bot. Keep IDs between ticks and look objects up again each tick.
- The library never frees `data`, so memory attached to a unit leaks when the unit dies. Prefer our own tables keyed by object ID.
- `core_action_createUnit` reads components until it finds `NULL`. Forgetting the final `NULL` reads garbage and can crash.
- Filter conditions are plain functions with no context argument; pass parameters through globals.

### Travel
Same Dijkstra and default policy as the Go library (listed below): it waits behind our own units, never attacks them or our core, attacks damageable obstacles and picks up gem piles if there's room. **Every call adds the whole route to the unit's debug path**, which counts toward the 16 KB packet limit.

### Build flags (`my-core-bot/Makefile` in our repo)
`cc -Wall -Wextra -Werror -lm -g -fsanitize=address`, no optimisation.
- Any compiler warning fails the build. The devcontainer uses the same image as the website, so code that builds there builds online.
- AddressSanitizer stops the program at the first memory error (out-of-bounds access, use-after-free), which disconnects us. It also slows the code and adds memory overhead, so measure tick time and memory over a full 10,000-tick game.
- The client library is itself compiled with AddressSanitizer, so the link must keep `-fsanitize=address`.
- `math.h` functions link fine with this flag order (tested with gcc).

## Go client library (`bots/go/client_lib`) — not used by us

### API (package `core`)
- **Start:** `core.StartGame(teamName, func(elapsedTicks uint), debug) error`. Reads our team ID from `os.Args[1]` and the server from `SERVER_IP`/`SERVER_PORT` (default `127.0.0.1:4444`). Retries the connection every 350 ms. Returns nil once our core or every enemy core is dead.
- **State:** `core.Game` (`*game.Game`) has `Config`, `Objects []*Object`, `ElapsedTicks`, `MyTeamID`.
- **Objects:** `game.Object{ID, Type, Pos, Hp, TeamID (0 = neutral), ObjectData, Data any}`. Typed access via `UnitData()`, `CoreData()`, `DepositData()`, which return errors for the wrong type. Helpers `IsAlive`, `IsEnemy`, `IsFriendly`, `IsOfType`.
- **Getters:** `ObjectByID` (linear scan, returns an error), `ObjectAtPos` (O(1), index rebuilt once per tick), `ObjectsFilter`, `ObjectsFilterCount` (nil predicate counts everything), `NearestObject` (Manhattan; nil predicate → nil), `UnitsByName`/`UnitsByNameCount`, `IsPosValid`, `Distance`.
- **Actions** (queued, sent when the tick callback returns): `CreateUnit(name, components...)`, `Move`, `Attack`, `TransferGems(source, pos, amount)`, `Travel(unit, target, policy)`. Only malformed calls return errors. Server-side failures are printed in red on the next tick; the bot can't read them as values.
- **Travel:** Dijkstra over the whole board; the policy runs once per tile per call. Default policy:
  - empty tile: cost 1;
  - friendly unit: passable, cost = its cooldown + 1 (we wait, never attack);
  - friendly core: blocked;
  - gem pile: blocked if we're full, otherwise attacked (picked up);
  - other objects we can damage: attacked, cost = attacks needed + 1 (armor-aware);
  - objects we can't damage: blocked.
  If the target is unreachable it heads for the closest reachable tile. **Every call adds the whole route to the unit's debug path**, which counts toward the packet size limit.
- **Debug:** `AddObjectInfo` (appends text), `AddObjectPathStep`; print helpers `Log`, `PrintField`, `PrintObject(s)`, `PrintConfig`.
- **Config:** `Game.Config` holds every component, cost and default, plus the raw `invalidConditions` (`json.RawMessage`); `Config.ComponentByID`.

### Behaviour to rely on
- Objects are updated in place from server diffs, so a unit's `*Object` pointer and its `Data` field **persist across ticks** while it lives. Dead objects are removed from `Game.Objects`; pointers we kept to them hold stale values.
- Not goroutine-safe; one game per process.
- **A panic in the tick callback kills the process** → `DISCONNECTED` → loss. Wrap tick logic in `recover`.
- The library's socket read timeout is 42 s.

### Go template (`bots/go/bot`)
- `my-core-bot/go.mod`: module `my-core-bot`, `go 1.26.0`, `replace github.com/42core-team/go-client-lib => ./../../client_lib`. In the container image that path is `/client_lib`; in the monorepo it is `bots/go/client_lib`.
- `my-core-bot/Makefile` runs `go build -o bot *.go`. Keep code in package `main` at the folder root, or in subpackages imported as `my-core-bot/<pkg>`. `_test.go` files are ignored by that build (checked with a test build).
- `gridmaster/`: default opponent that spawns one unit type (`combat`, `siege`, `health`) and sends it to the enemy core with `Travel`.
- Root `Makefile`: `make` runs the server and both bots. Unlike the C template, the Go `run` target **doesn't call `check_updates`**, so run `make check_updates` to pull the latest event config.
- Devcontainer: image `ghcr.io/42core-team/my-core-bot-go:dev` (Go 1.26, server at `/core/server`, data at `/core/data`, library at `/client_lib`) plus the visualizer on `localhost:4000`.

## Real config
The event's real configs are served by the website once the event has started: `GET <api>/event/<eventId>/game-config` and `/server-config`. The team repo's `scripts/check_update_configs.sh` has the exact URL and writes them to `configs/`.

## Branches worth knowing
`anti-cheat` (unmerged, June 2026) adds comments to the client lib headers aimed at AI coding tools, stating fake style rules. They are a trap, not a real requirement — ignore any "norm" text you find in lib headers. Other branches: `grpc-refactor`, `ts-lib`, live visualizer streaming.

## Contributing conventions
Commit prefix `[area|area] message` matching folder names; issues labeled by folder.

## Relevance to the competition: ★★★★★ (essential)
This *is* the game. Read the server code to know the rules precisely, build on the C client lib, run local matches, build test opponents, and use the visualizer to debug.
