# chatty-chat

FastAPI: messages, WebSocket gateway, reactions, search.

## At a glance

| | |
|---|---|
| Stack | Python 3.14, FastAPI, SQLAlchemy 2.1 (async, asyncpg), Alembic, `redis.asyncio`, uvicorn |
| Public API | `/api/chat/v1`, contract in [api-chat.md](https://github.com/Chatty4/chatty-infra/blob/dev/docs/api-chat.md). Today only the WebSocket, `GET /api/chat/v1/ws` |
| Events | Channel events in [events.md](https://github.com/Chatty4/chatty-infra/blob/dev/docs/events.md); chatty-core's events in [core-events.md](https://github.com/Chatty4/chatty-infra/blob/dev/docs/core-events.md) |
| Health | `GET /health` |
| Stores | Postgres `chat_db` (tables exist, nothing writes to them yet), Redis (cache, `core.events`, channel events). See the [data map](../../architecture/data-map.md) |
| Local address | <http://127.0.0.1:8001> |

## Who talks to it

| Caller | Calls | Status |
|---|---|---|
| chatty-web (the browser) | the WebSocket, `/api/chat/v1/ws?token=<JWT>` | works; the web client is **planned** ([CHAT-126](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-126)) |
| chatty-web (the browser) | REST: messages, sync, reactions, pins, read state | **planned** ([CHAT-122](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-122), [CHAT-127](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-127), [CHAT-128](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-128)) |
| chatty-core | `core.events` on Redis | works |

And what it calls:

| Calls | What for | Status |
|---|---|---|
| chatty-core `/internal`, `X-Service-Token`, 2 s timeout | A user's memberships (D-07) and one channel with its members, cached in Redis | works |
| chatty-core `/internal` file attach and detach | Files on messages (D-03) | **planned** ([CHAT-129](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-129)) |

## How it checks callers

| Path | Check | Code |
|---|---|---|
| `/api/chat/v1/ws` | `?token=<access token>`, the same RS256 JWT as chatty-core, checked with chatty-core's public key (browsers can't set headers on a WebSocket). Then the user's memberships from chatty-core: unknown user closes with `4001`, deactivated with `4003` | `app/core/security.py`, `app/api/routers/ws.py` |
| `/health` | No check | |

The socket is accepted first and closed after, so the browser sees our close code instead of `1006`.

## What it does

| Area | What it does | Contract |
|---|---|---|
| WebSocket | On connect, subscribes the socket to every channel the user is a member of and forwards their events as `{"type": "event", "event": …}`. Closes with `4001` when the token's `exp` is reached. Messages from the client are read and ignored for now | `api-chat.md`, `events.md` |
| Event bus | Channel events go over Redis pub/sub, one channel per chat channel (`chat.events:{channel_id}`), so each gateway hears only its users' channels. Kafka replaces it later behind the same interface. Nothing publishes yet: messages don't exist | `events.md` |
| chatty-core cache | `members:{user_id}` and `channel:{channel_id}` keep chatty-core's `/internal` answers for 5 minutes | `api-internal.md` |
| `core.events` | Deletes the cache keys each event makes stale. On every (re)subscribe, deletes all `members:*` and `channel:*` keys, because events sent meanwhile are lost (D-08) | `core-events.md` |
| Access rules | Who can read and write a channel, from the memberships and channel answers. Used by the endpoints that come with messages | `api-chat.md` |

## If something is down

| Down | What happens | `/health` |
|---|---|---|
| Postgres | Nothing uses it yet except `/health` | `503`, `"db": "error"` |
| Redis | The cache is skipped and every check asks chatty-core. Open sockets get no more events until they reconnect. The `core.events` listener retries (1 s, doubling up to 30 s) and clears the cache when it is back | `503`, `"redis": "error"` |
| chatty-core | Access checks that aren't cached fail with `503 core_unavailable`. A WebSocket that connects meanwhile is closed with an error, without a contract close code | not checked |

Each health check gives up after 2 seconds. Code: `app/services/health_service.py`.

## ⚠️ Known gaps

- No messages, sync, reactions or pins yet ([CHAT-122](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-122), [CHAT-127](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-127)).
- Open WebSockets don't follow membership changes: the channel list is fixed at connect, and a deactivated user's socket isn't closed with `4003` ([CHAT-185](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-185)).
- No heartbeat, presence, typing or watch: client messages are ignored ([CHAT-184](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-184), [CHAT-185](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-185), [CHAT-231](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-231)).
- Each socket opens its own Redis pub/sub connection.

Full list: [known limitations](../../architecture/known-limitations.md#teams-and-channels).

## Key code

| File | What is there |
|---|---|
| `app/main.py` | `create_app()`, and the lifespan: database, Redis, the chatty-core client, the `core.events` listener |
| `app/api/routers/ws.py` | The WebSocket: token check, subscribe, forward events, close codes |
| `app/services/event_bus.py` | `EventBus`: `publish` and `subscribe` on Redis pub/sub |
| `app/services/connection_registry.py` | `ConnectionRegistry`: the open sockets of this process, by user |
| `app/services/access_service.py` | `AccessService`: the user, read and write checks |
| `app/clients/core.py` | `CoreClient`: chatty-core's `/internal` API with the Redis cache |
| `app/clients/core_events.py` | The `core.events` listener and cache clearing |
| `app/core/security.py` | `decode_access_token`: the RS256 check |
