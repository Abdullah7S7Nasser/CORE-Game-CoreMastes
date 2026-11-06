# 3d-visualizer — 3D replay viewer (archived)

**Status:** ARCHIVED ("development stopped"). 60 commits, Feb 2025 → Sep 2025.
**Stack:** SvelteKit, Threlte (Three.js for Svelte), TypeScript, ESLint/Prettier, Husky.

## What it is
An experimental 3D visualizer for replays: grid environment, wall blocks, core/unit/tree sprites, settings panel.

## Contents
- `src/lib/replayManager.ts` — loads `static/replays/replay_latest.json`.
- `src/lib/components/` — `Game.svelte`, `Settings.svelte`, `blocks/WallBlock.svelte`, `environment/Grid.svelte`, `sprites/{Core,Unit,Tree}.svelte`.
- `src/routes/+page.svelte` — main page.

## Run
`npm install && npm run dev`.

## Relevance to the competition: ☆☆☆☆☆
Abandoned, likely incompatible with the current replay format. Not needed.
