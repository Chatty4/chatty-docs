# Chatty Docs

How Chatty works. Written for someone who has never seen the code.

> **In one minute:** Chatty is a team chat: teams, channels, DMs and live messages.
> The backend is two services: chatty-core (users, auth, teams, channels, files) and chatty-chat (messages and the live WebSocket gateway).
> This repo explains what each service does and how they work together.

## Start here

1. Read [System overview](architecture/overview.md) for the big picture. (5 min)
2. Pick a [service page](#services) to see how one repo works.
3. Pick a [flow](architecture/flows/README.md) to follow one full journey, for example "change password".

## This repo vs chatty-infra

| Question | Where |
|---|---|
| What exactly does an endpoint accept and return? | [chatty-infra/docs](https://github.com/Chatty4/chatty-infra/tree/dev/docs): `api-core.md`, `api-chat.md`, `api-internal.md` |
| What events exist and what is in them? | chatty-infra/docs: `events.md`, `core-events.md` |
| What did the team agree on? | chatty-infra/docs: `decisions.md` |
| How does it work, and why? | **here** |

The contracts are the source of truth. Pages here link to them and never copy endpoint tables.

## Pages

| Page | What you learn |
|---|---|
| [System overview](architecture/overview.md) | All services, the diagram, how they talk, auth in 30 seconds |
| [Auth and security](architecture/auth-and-security.md) | Tokens, refresh rotation, the service token, passwords, keys |
| [Live events](architecture/realtime.md) | Sockets, Redis pub/sub, gateways and the cache: how a change reaches every open tab |
| [Data map](architecture/data-map.md) | What is stored in Postgres, Redis, MinIO and Kafka, and why |
| [Decisions](architecture/decisions.md) | The reasons behind D-01 to D-11, in plain words |
| [Glossary](architecture/glossary.md) | Plain meanings for the hard words |
| [Known limitations](architecture/known-limitations.md) | What is not right yet, with links |
| [Flows](architecture/flows/README.md) | One user journey per page |

## Services

| Service | Repo | Page |
|---|---|---|
| Core (Django) | [chatty-core](https://github.com/Chatty4/chatty-core) | [services/core](services/core/index.md) |
| Chat (FastAPI) | [chatty-chat](https://github.com/Chatty4/chatty-chat) | [services/chat](services/chat/index.md) |
| Web (React) | [chatty-web](https://github.com/Chatty4/chatty-web) | [services/web](services/web/index.md) |
| Infra (Docker Compose) | [chatty-infra](https://github.com/Chatty4/chatty-infra) | [services/infra](services/infra/index.md) |

## Writing rules

- Describe only what exists in code. Mark planned work as **planned** and link the Jira epic.
- Picture first, then numbered steps, then "If a step fails".
- Link to code as `repo/path/file.py`, not by copying it.
