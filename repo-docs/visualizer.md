# visualizer — CORE v1 live visualizer (abandoned)

**Status:** ABANDONED (v1). 105 commits, Dec 2023 → May 2025.
**Stack:** Go (websocket/socket bridge), p5.js front end (`public/sketch.js`, ~1,000 lines), Docker.

## What it is
The first visualizer. A Go server connects to the v1 game server's socket as a spectator and forwards state over WebSockets to a browser page that draws the game with p5.js.

## Contents
- `main.go`, `socket/socket.go` (game server connection), `websockethandler/websocket.go`, `env/environmentvariables.go`.
- `public/` — `index.html`, `sketch.js`, `style.css`, sample `config.json`/`state.json`, images and joke sound effects in `assets/archive`.

## Relevance to the competition: ☆☆☆☆☆
Built for the Rust v1 server protocol. Not usable with the current game.
