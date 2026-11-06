# debug-visualizer — 2D replay viewer (archived)

**Status:** ARCHIVED (now `monorepo/visualizer`). 169 commits, Jun 2025 → Aug 2025.
**Stack:** TypeScript (vanilla), Vite, Biome, Docker + nginx, Helm chart, multi-arch GHCR images.

## What it is
A browser app that loads a replay JSON produced by the server and plays it back: play/pause/reverse, step tick/action, speed, skip to start/end. Shows units, cores, deposits, walls and gem piles with tooltips.

## Architecture
1. **Time manager** — playback state; produces `tickData {gameTick, animTick, tickfocused}`.
2. **Replay loader** — applies the server's state diffs to give full state at any tick.
3. **Displayer/renderer** — renders frame from tick data; animations are math functions of `animTick`.

Files: `src/ts/input_manager/`, `src/ts/renderer/`, `src/ts/replay_loader/`, `public/misc/replay_latest.json`, `helm/debug-visualizer/`.

## Run
`npm run dev` (dev), `npm run build`, `npm run serve` (port 8080), or `docker run -p 8080:80 debug-visualizer`.

## Relevance to the competition: ★☆☆☆☆
Use the monorepo visualizer (it has the newer component icons, debug paths, tooltips). The replay-loader code here is still a good reference if you want to write your own replay-analysis script.
