# cli — organizer TUI (legacy)

**Status:** LEGACY (last commit Mar 2025). 92 commits. README is empty.
**Stack:** Go, Charm TUI libraries (Bubble Tea, Lip Gloss) in `tui/`, Docker SDK, GitHub API, SQLite via GORM.

## What it is
An admin tool for running early CORE events from a terminal: manage teams/players, create or delete team GitHub repos from a template, and launch matches as Docker containers (server + bot-clients + visualizer).

## Contents
- `tui/` — screens for main menu, teams, players, repos, games, spinner.
- `github/` — client, repos (create/delete all), users.
- `docker/` — containers, images, networks, logs, cleanup.
- `game/` — `RunGame` (team vs team), `RunGameSpectator`, `RunGameAgainstStarlord` (vs a house bot repo `42-core-repos/starlord`).
- `db/`, `model/`, `logging/`.
- `.env.example`: `GITHUB_TOKEN`, `GITHUB_ORG`, `STARLORD_REPO`, `SEED`, `TICK_RATE`, image names on `registry.coregame.de`.

## Relevance to the competition: ☆☆☆☆☆
Organizer tooling, replaced by the website. Not useful for building a bot.
