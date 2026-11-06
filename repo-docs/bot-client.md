# bot-client — bot build/run container (legacy infra)

**Status:** LEGACY (last commit Apr 2025; replaced by the website's `k8s-service`). 30 commits.
**Stack:** Go, RabbitMQ (amqp), Docker image built in CI for self-hosted x64 runners.

## What it is
A small Go program run inside a container for each bot in an online match: clone the team repo, build the bot, run it, and report status/logs to RabbitMQ.

## Flow (`main.go`)
`git.Clone(REPO_URL)` → `exec.BuildBot()` → `exec.RunBot()`; errors via `fail/`, logs/status via `rabbitmq/` (`botlog.go`, `status.go`).

## Config (`.env-example`)
`REPO_URL`, `GIT_ACCESS_TOKEN`, `PLAYER_ID`, `CLIENT_ID`, `ENABLE_RABBITMQ`, `RABBITMQ_URL`.

## Relevance to the competition: ☆☆☆☆☆
Infrastructure only. It does confirm the long-standing rule: your repo is cloned, `make` is run, and the binary is executed with your player id.
