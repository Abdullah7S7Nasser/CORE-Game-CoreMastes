# website — coregame.sh platform

**Status:** ACTIVE. 2,097 commits, last commit 2026-11-06.
**Stack:** React 19 + TanStack Start + Vite + Tailwind 4 (frontend); NestJS 12 + TypeORM + PostgreSQL (API); NestJS GitHub microservice; Go Kubernetes service; RabbitMQ; SeaweedFS (S3). Local dev via k3d + Tilt + Helm.

## What it is
The competition platform at https://coregame.sh: sign-in (GitHub / 42 OAuth), events, teams, auto-created team repos, the **Unit Builder**, the matchmaking queue, Swiss + elimination tournament, replays, logs, betting ("gambling" credits), stats, and an embedded wiki.

## Services
| Service | Role |
| --- | --- |
| `frontend/` | Routes: `events/$id/...` (incl. `unit-builder`), `wiki`, `profile`, `rush`, `changelog`. Component `components/unit-builder/UnitBuilder.tsx`. |
| `api/` | Modules: `auth`, `event`, `team`, `match`, `gambling`, `media`, `stats`, `user`, `github-api`. |
| `github-service/` | Creates/validates team repos from templates, manages permissions, encrypts secrets. |
| `k8s-service/` | Go. Consumes `game_queue` from RabbitMQ, clones repos, starts a game pod (server + 2 bot containers + iptables sidecar), uploads replays/logs to S3. |
| `docs/adr/` | ADRs: event lifecycle, session refresh tokens, GitHub grant check. |

## Event lifecycle
Registration → Coding (`PLAYING`) → Code freeze (`SUBMISSIONS_CLOSED`, repos locked) → Showtime (`SHOWCASE`) → Completion. See `GLOSSARY.md`.

## How matches are judged (`TOURNAMENT_LOGIC.md`, `api/src/match/match.service.ts`)
- **Queue:** random opponents near in time; standard Elo on `queueScore` (K = 32, floor 0). Results not revealed publicly.
- **Swiss phase:** `ceil(log2 N)` rounds, 1 point per win, no rematches, odd count → one bye (= win, max once).
  Tie-break = **modified Buchholz: sum of current scores of opponents you *beat*** (losses give nothing).
- **Advancement:** top `2^floor(log2 N)` teams (optionally capped by admin limit) go to single elimination.
- **Elimination:** seeded snake bracket (1st and 2nd seeds can only meet in the final).

## How our bot runs online (`k8s-service/internal/kube/game.go`)
- An init container clones the team repo into `/shared-data`. The bot container then runs:
  `mkdir -p /shared-data/.cache /tmp/go/pkg/mod /tmp/go/build-cache && cd /shared-data/my-core-bot && make && ./bot <rndId>`
  (the wiki says `/shared-data/repo/my-core-bot`; the code uses `/shared-data/my-core-bot`).
- **The build happens at match start**, so compile time counts against the server's connect timeout. `GOCACHE`/`GOMODCACHE` live in an empty `/tmp`, so every match is a cold Go build.
- Env: `HOME=/shared-data`, `GOPATH=/tmp/go`, `GOCACHE=/tmp/go/build-cache`, `GOMODCACHE=/tmp/go/pkg/mod`, `TMPDIR=/tmp`.
- Writable: `/shared-data` and `/tmp` (250 Mi emptyDir each). Root filesystem read-only, UID 2000, non-root, all capabilities dropped.
- Bot resources: request 250m CPU / 256 Mi, **limit 1 CPU / 512 Mi**. Game server: limit 2 CPU / 1 Gi.
- A `net-guard` init container blocks all non-loopback egress for the bot UID, so builds can't download modules.

## Public config endpoints (`api/src/event/event.controller.ts`)
- `GET /event/:id/game-config` and `/event/:id/server-config` return the event's real configs (null before the start date). The team repo's `scripts/check_update_configs.sh` calls them.
- `GET /event/:id/version` (and `/templates/:templateId/version`) returns the expected template and visualizer image versions, used by `check_image_updates.sh`.

## Run locally
```bash
make          # creates k3d cluster core-dev, asks for OAuth creds, starts Tilt
# frontend :3000, api :4000, rabbitmq :15672, seaweedfs :9000-9002, k8s-service :9003, postgres :5432
```

## Relevance to the competition: ★★★☆☆ (useful context, not code you'll run)
You won't modify it, but it tells you the scoring rules (every win counts; beating strong teams matters for Buchholz), how your bot is built and run online and under which limits, where to fetch the real event config, and that the **Unit Builder** on the event page shows the component IDs.
