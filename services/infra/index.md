# chatty-infra

Docker Compose, contracts, setup scripts.

## At a glance

| | |
|---|---|
| Runs | `docker-compose.yml`: core-db, chat-db, redis, minio, kafka, core, chat |
| Contracts | `docs/`: `api-core.md`, `api-chat.md`, `api-internal.md`, `core-events.md`, `events.md`, `decisions.md` |
| Scripts | `setup.bat`, `start.bat`, `stop.bat`, `restart.bat`, `migrate.bat`, `logs.bat`, `status.bat`, `test.bat`, `shell.bat`, `reset-db.bat`; `chatty.bat` is a menu for all of them |
| Postman | `postman/`: a collection with every HTTP endpoint of chatty-core (public and `/internal`) and chatty-chat, and a local environment. The WebSocket can't be stored in a collection: open it in Postman with New > WebSocket |

How to start and which ports: the [chatty-infra README](https://github.com/Chatty4/chatty-infra#readme).

## Who talks to it

Developers, not services. Every service reads its `.env` (created by `setup.bat`) and connects to the
containers this repo starts.

## How it checks callers

It doesn't. Every port is published on `127.0.0.1` only, for local development. `setup.bat` creates the
shared secrets once: `CORE_SERVICE_TOKEN`, chatty-core's `SECRET_KEY`, and the JWT key pair in
`chatty-core/keys/` (see [auth and security](../../architecture/auth-and-security.md#secrets-and-keys)).

## What it does

| Area | What it does |
|---|---|
| First setup | `setup.bat` clones the other repos on `dev`, creates every `.env` with matching secrets, creates the JWT keys once, creates the VS Code workspace and can set up virtualenvs. Safe to run again |
| Daily | Start, stop, restart and rebuild containers; follow logs; check health and branches |
| Databases | Migrations for core and chat; `reset-db.bat` backs up to `backups\` before it wipes anything |
| Tests | `test.bat` runs the same checks as CI (ruff, pytest) inside the containers |
| Contracts | The single source of truth for APIs, events and decisions. Change them here first, in their own PR |

## If something is down

| Problem | Fix |
|---|---|
| A container isn't healthy | `status.bat`, then `logs.bat <service> --errors` |
| Changed `.env` or compose | `restart.bat <service>` recreates it |
| Changed `requirements.txt` | `restart.bat <service> --build` |
| Connections stall on Windows | Use `127.0.0.1`, never `localhost` (Windows tries IPv6 first) |

## ⚠️ Known gaps

- No HTTPS and no Nginx in the dev stack. See [known limitations](../../architecture/known-limitations.md#infra).
- chatty-web isn't in compose; it runs with its own dev server.

## Key code

| File | What is there |
|---|---|
| `docker-compose.yml` | Every container, its ports, volumes and health check |
| `setup.bat` | First-time setup, step by step |
| `scripts/lib.bat` | Shared helpers for the scripts |
| `scripts/check-env-example.sh` | Checks that `.env.example` and `docker-compose.yml` use the same variables |
| `docs/` | The contracts |
