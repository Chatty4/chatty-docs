# chatty-chat

FastAPI: messages, the WebSocket gateway, sync. Later reactions, pins, files and search.

## At a glance

| | |
|---|---|
| Stack | Python 3.14, FastAPI, SQLAlchemy 2.1 (async, asyncpg), Alembic, `redis.asyncio`, uvicorn |
| Public API | `/api/chat/v1`, contract in [api-chat.md](https://github.com/Chatty4/chatty-infra/blob/dev/docs/api-chat.md): messages, history, sync, channel events, the WebSocket `GET /api/chat/v1/ws` |
| Events | Channel events in [events.md](https://github.com/Chatty4/chatty-infra/blob/dev/docs/events.md); chatty-core's events in [core-events.md](https://github.com/Chatty4/chatty-infra/blob/dev/docs/core-events.md) |
| Health | `GET /health` |
| Stores | Postgres `chat_db` (messages, channel events, seq counters, read state), Redis (cache, `core.events`, `chat.events:*`). See the [data map](../../architecture/data-map.md) |
| Local address | <http://127.0.0.1:8001> |

How the live parts work together: [live events](../../architecture/realtime.md).

## Who talks to it

| Caller | Calls | Status |
|---|---|---|
| chatty-web (the browser) | REST: send, edit, delete, history, `POST /sync`, channel events | works; the web client is **planned** ([CHAT-126](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-126)) |
| chatty-web (the browser) | the WebSocket, `/api/chat/v1/ws?token=<JWT>` | works |
| chatty-web (the browser) | REST: reactions, pins, read state, attachments, search | **planned** ([CHAT-127](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-127), [CHAT-128](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-128)) |
| chatty-core | `core.events` on Redis | works |

And what it calls:

| Calls | What for | Status |
|---|---|---|
| chatty-core `/internal`, `X-Service-Token`, 2 s timeout | A user's memberships (D-07) and one channel with its members, cached in Redis | works |
| chatty-core `/internal` file attach and detach | Files on messages (D-03) | **planned** ([CHAT-212](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-212)) |

## How it checks callers

| Path | Check | Code |
|---|---|---|
| `/api/chat/v1/*` (REST) | `Authorization: Bearer <access token>`, the same RS256 JWT as chatty-core, checked with chatty-core's public key. Then the user's memberships: unknown user `401`, deactivated `403 user_inactive` | `app/api/deps.py`, `app/core/security.py` |
| `/api/chat/v1/ws` | `?token=<access token>` (browsers can't set headers on a WebSocket), hidden as `token=***` in the logs. Unknown user or bad token closes with `4001`, deactivated with `4003` | `app/api/routers/ws.py`, `app/core/logging.py` |
| `/health` | No check | |

The socket is accepted first and closed after, so the browser sees our close code instead of `1006`.

**Who can do what** (rules in `app/services/access_service.py`, from the memberships and channel answers):

| Action | Who |
|---|---|
| Read (history, events, sync) | Channel members; anyone in the team for a public channel |
| Send, edit | Channel members, while the channel isn't archived and its team isn't deleted. Edit: the author only |
| Delete | The author; outside DMs also the channel owner and team owners and admins (D-05) |

## What it does

| Area | What it does | Flow |
|---|---|---|
| Send | One transaction: the channel's next `seq` (a locked counter row), the message, a `channel_events` row, the sender's read position. Publishes `message.created` after the commit. A retry with the same `client_msg_id` returns the original with `200` | [send a message](../../architecture/flows/send-message.md) |
| Edit, delete | Edit the body (author only) or soft-delete (D-05). Both take the next `seq` and publish `message.edited` / `message.deleted` | [edit and delete](../../architecture/flows/edit-and-delete-message.md) |
| History | `GET /channels/{id}/messages`: newest first, by `before_seq`, with the channel's `last_seq`. Deleted messages stay, empty | [reconnect and sync](../../architecture/flows/reconnect-and-sync.md) |
| Sync | `POST /sync` for many channels and `GET /channels/{id}/events` for one: the events after a `seq`, or `gap_too_large` | [reconnect and sync](../../architecture/flows/reconnect-and-sync.md) |
| WebSocket gateway | One Redis pub/sub connection per process, one subscription per channel with a local socket. Forwards channel events, follows membership changes, closes deactivated users, heartbeat and watch | [connect the WebSocket](../../architecture/flows/websocket.md), [live events](../../architecture/realtime.md) |
| Event bus | `EventBus.publish`: one `PUBLISH` on `chat.events:{channel_id}` per event, after the commit. Kafka replaces it later | [live events](../../architecture/realtime.md#from-a-saved-change-to-every-tab) |
| chatty-core cache | `members:{user_id}` and `channel:{channel_id}` for 5 minutes. A write is refused if an event bumped the key's version meanwhile (CHAT-266) | [live events](../../architecture/realtime.md#the-cache-and-its-versions) |
| `core.events` | Marks the cache keys an event makes stale; the gateway updates open sockets. On every (re)subscribe the whole cache is cleared (D-08) | [core events](../../architecture/flows/core-events.md) |

## If something is down

| Down | What happens | `/health` |
|---|---|---|
| Postgres | Every message, history and sync call fails with `500` | `503`, `"db": "error"` |
| Redis | The cache is skipped and every check asks chatty-core. Publishing is logged and skipped; clients catch up with `POST /sync`. When Redis is back, the gateway closes every socket with `1001` and the cache is cleared | `503`, `"redis": "error"` |
| chatty-core | Access checks that aren't cached fail with `503 core_unavailable`. A WebSocket that connects meanwhile is closed with `1011` | not checked |

Each health check gives up after 2 seconds. Code: `app/services/health_service.py`.

## ⚠️ Known gaps

- No reactions, pins, read state endpoint, unread counts or files yet ([CHAT-127](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-127), [CHAT-128](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-128), [CHAT-212](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-212)).
- No typing or presence on the WebSocket yet ([CHAT-215](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-215)).
- Events go over Redis pub/sub only: the outbox relay to Kafka and the 30-day cleanup of `channel_events` don't run yet.
- Watch uses the memberships loaded at connect ([CHAT-231](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-231)).

Full list: [known limitations](../../architecture/known-limitations.md#messages-and-live-events).

## Key code

| File | What is there |
|---|---|
| `app/main.py` | `create_app()`, and the lifespan: database, both Redis clients, the chatty-core client, the gateway and the `core.events` listener |
| `app/api/routers/messages.py` | Send, history, edit, delete |
| `app/api/routers/sync.py` | `POST /sync`, `GET /channels/{id}/events` |
| `app/api/routers/ws.py` | The WebSocket: token check, client messages, deadlines, close codes |
| `app/services/message_service.py` | `MessageService`: send, edit, delete, history, publish after commit |
| `app/services/sync_service.py` | `SyncService`: sync and channel events, the gap checks |
| `app/services/gateway.py` | `Gateway`: one pub/sub connection, fan-out to local sockets, core events, watch |
| `app/services/connection_registry.py` | `ConnectionRegistry`: the open sockets of this process, by user and by channel |
| `app/services/event_bus.py` | `EventBus.publish` on Redis pub/sub |
| `app/services/access_service.py` | `AccessService`: the user, read, write and delete checks |
| `app/repositories/` | Messages, channel events, the seq counter, read state |
| `app/clients/core.py` | `CoreClient`: chatty-core's `/internal` API with the versioned Redis cache |
| `app/clients/core_events.py` | The `core.events` listener and cache clearing |
| `app/core/security.py` | `decode_access_token`: the RS256 check |
