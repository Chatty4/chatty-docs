# chatty-chat

FastAPI: messages and their files, reactions, pins, presence, search, read state, push notifications, the
WebSocket gateway, sync.

## At a glance

| | |
|---|---|
| Stack | Python 3.14, FastAPI, SQLAlchemy 2.1 (async, asyncpg), Alembic, `redis.asyncio`, uvicorn |
| Public API | `/api/chat/v1`, contract in [api-chat.md](https://github.com/Chatty4/chatty-infra/blob/dev/docs/api-chat.md): messages, attachment download links, reactions, pins, history, sync, channel events, presence, read state and unread counts, search, push subscriptions, the WebSocket `GET /api/chat/v1/ws` |
| Events | Channel events in [events.md](https://github.com/Chatty4/chatty-infra/blob/dev/docs/events.md); chatty-core's events in [core-events.md](https://github.com/Chatty4/chatty-infra/blob/dev/docs/core-events.md) |
| Health | `GET /health`: the database, Redis and the two background tasks. The relay and the push worker have no health check |
| Processes | The API (`app.main:app`), the outbox relay (`python -m app.jobs.relay`) and the push worker (`python -m app.jobs.push_worker`), each in its own container from the same image |
| Stores | Postgres `chat_db` (messages, attachments, reactions, pins, channel events, seq counters, read state, push subscriptions), Redis (cache, presence, push dedupe, `core.events`, `chat.events:*`, `chat.typing:*`, `chat.presence:*`, `chat.users:*`), Kafka (`chat.events`). See the [data map](../../architecture/data-map.md) |
| Local address | <http://127.0.0.1:8001> |

How the live parts work together: [live events](../../architecture/realtime.md).

## Who talks to it

| Caller | Calls | Status |
|---|---|---|
| chatty-web (the browser) | REST: send (with files), edit, delete, reactions, pins, history, `POST /sync`, channel events, presence, `POST /channels/{id}/read`, `GET /unread`, `GET /search`, `GET /messages/{id}/attachments` | works |
| chatty-web (the browser) | the WebSocket, `/api/chat/v1/ws?token=<JWT>` | works |
| chatty-web (the browser) | `POST` / `DELETE /push/subscriptions` | works |
| chatty-core | `core.events` on Redis | works |

And what it calls:

| Calls | What for | Status |
|---|---|---|
| chatty-core `/internal`, `X-Service-Token`, 2 s timeout | A user's memberships (D-07) and one channel with its members, cached in Redis | works |
| chatty-core `/internal` file attach and detach | Files on messages (D-03), never cached | works |
| Nothing (signing only) | MinIO download links for attachments, with chatty-chat's own read-only key (D-02, D-16). Signing makes no network call | works |
| Kafka `chat.events` | The relay produces channel events, the push worker consumes them | works |
| Browser push services (Google, Mozilla, Apple, Windows), 5 s timeout | Web Push from the push worker, signed with the VAPID private key | works |

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
| React | Channel members, while the channel isn't archived and its team isn't deleted; only their own reactions (D-05) |
| Pin, unpin | Channel members, while the channel isn't archived and its team isn't deleted. Any of them can unpin, not only who pinned. Listing pins: anyone who can read |
| Typing | Channel members, while the channel isn't archived and its team isn't deleted |
| See who is online | Members of the team |

## What it does

| Area | What it does | Flow |
|---|---|---|
| Send | Up to 10 files are attached in chatty-core first (D-03). Then one transaction: the channel's next `seq` (a locked counter row), the message, its attachments, a `channel_events` row, the sender's read position. Publishes `message.created` after the commit. A retry with the same `client_msg_id` returns the original with `200`. Files of a failed send are detached | [send a message](../../architecture/flows/send-message.md) |
| Attachments | `GET /messages/{id}/attachments`: download links for all the files of a message, in the order sent, signed by chatty-chat itself, valid 5 minutes (D-02) | [send a message](../../architecture/flows/send-message.md) |
| Edit, delete | Edit the body (author only) or soft-delete (D-05). Both take the next `seq` and publish `message.edited` / `message.deleted`. A delete also clears the text in the message's earlier events, deletes its reactions and pin, and detaches its files after the commit | [edit and delete](../../architecture/flows/edit-and-delete-message.md) |
| Reactions | `PUT` / `DELETE /messages/{id}/reactions/{emoji}`: add or remove my emoji. A real change takes the next `seq` and publishes `reaction.added` / `reaction.removed`; the same add twice, or removing one I don't have, is `200` with no event. Both answer the message's reactions with `count` and `me` | [react to a message](../../architecture/flows/react-to-a-message.md) |
| Pins | `POST` / `DELETE /channels/{id}/pins/{message_id}` and `GET /channels/{id}/pins`. A pin takes the next `seq` and publishes `pin.added`; pinning again is `200` with the existing pin and no event. Max 100 per channel (`409 pin_limit_reached`), checked under the counter lock so two pins can't both pass at 99. Unpinning one that isn't pinned is `204` with no event. The list is newest first, with full message objects | [pin a message](../../architecture/flows/pin-a-message.md) |
| Presence | A user is online while they have an open WebSocket, in every team they're in. Redis keeps one line per socket, so the first socket sends `online` and the last sends `offline`, across processes. `GET /teams/{team_id}/presence` gives a new tab the current list | [see who is online](../../architecture/flows/see-who-is-online.md) |
| History | `GET /channels/{id}/messages`: newest first, with the channel's `last_seq`, each message's reactions, `pinned` and files (one query each for the page). `before_seq` pages back, `after_seq` pages forward (`has_more_after`), and `around_seq` centers a page on one message, to jump to it. All in one `REPEATABLE READ` snapshot. Deleted messages stay, empty | [reconnect and sync](../../architecture/flows/reconnect-and-sync.md) |
| Search | `GET /search?team_id=…&q=…`: Postgres full-text search (`english`) over the messages of every channel of the team I can read, optionally `in` one channel, `from` one author, `before` / `after` a time. Best match first, then newest, with an opaque cursor tied to the filters. Each result is the message object and a `highlight` with `<mark>` around the matched words, HTML-escaped. Deleted messages and mentions' IDs are never matched | [search messages](../../architecture/flows/search-messages.md) |
| Sync | `POST /sync` for many channels and `GET /channels/{id}/events` for one: the events after a `seq`, or `gap_too_large`. Channels the cache doesn't know are checked against chatty-core 20 at a time | [reconnect and sync](../../architecture/flows/reconnect-and-sync.md) |
| Typing | A `typing` message on the WebSocket, at most every 3 s per socket and channel, goes to the channel's other sockets through `chat.typing:{channel_id}`. Never stored, no `seq` | [live events](../../architecture/realtime.md#someone-is-typing) |
| WebSocket gateway | One Redis pub/sub connection per process, one subscription per channel with a local socket. Forwards channel events, typing and presence, follows membership changes, closes deactivated users, heartbeat and watch | [connect the WebSocket](../../architecture/flows/websocket.md), [live events](../../architecture/realtime.md) |
| Event bus | `EventBus.publish(ChannelEvent)`: one `PUBLISH` on `chat.events:{channel_id}` per event, after the commit, for the gateways | [live events](../../architecture/realtime.md#from-a-saved-change-to-every-tab) |
| Outbox relay | `python -m app.jobs.relay`, its own container: sends the unpublished `channel_events` rows to the Kafka topic `chat.events` in order, keyed by `channel_id`, then marks them. Woken by `NOTIFY`, polls every 0.2 s, one at a time (advisory lock) | [get a push notification](../../architecture/flows/get-a-push-notification.md) |
| Push | `POST` / `DELETE /push/subscriptions` save and forget this browser (D-13). `python -m app.jobs.push_worker`, its own container, reads `message.created` from Kafka and sends Web Push to members who want it and have no tab on screen | [get a push notification](../../architecture/flows/get-a-push-notification.md) |
| chatty-core cache | `members:{user_id}` and `channel:{channel_id}` for 5 minutes. A write is refused if an event bumped the key's version meanwhile (CHAT-266) | [live events](../../architecture/realtime.md#the-cache-and-its-versions) |
| Cleanup | `python -m app.jobs.clean_up_events` deletes `channel_events` older than 30 days, each channel's oldest first. Runs once a day from the `chat` container (CHAT-191). Clients that asked for deleted events get `gap_too_large` and reload the history | [reconnect and sync](../../architecture/flows/reconnect-and-sync.md) |
| `core.events` | Marks the cache keys an event makes stale; the gateway updates open sockets. On every (re)subscribe the whole cache is cleared (D-08) | [core events](../../architecture/flows/core-events.md) |

## If something is down

| Down | What happens | `/health` |
|---|---|---|
| Postgres | Every message, history and sync call fails with `500` | `503`, `"db": "error"` |
| Redis | The cache is skipped and every check asks chatty-core. Publishing is logged and skipped; clients catch up with `POST /sync`. When Redis is back, the gateway closes every socket with `1001` and the cache is cleared | `503`, `"redis": "error"` |
| chatty-core | Access checks that aren't cached fail with `503 core_unavailable`. A WebSocket that connects meanwhile is closed with `1011` | not checked |
| A background task (the gateway or the `core.events` listener) | It has stopped: open sockets or the cache are no longer kept right. The crash is logged | `503`, `"tasks": "error"` |
| Kafka | The relay retries every second and the rows stay unpublished; push notifications wait. Messages and live events are unaffected | not checked |
| MinIO | Nothing: download links are signed without a call; the browser gets the error when it follows one | not checked |

Each health check gives up after 2 seconds. Code: `app/services/health_service.py`.

## ⚠️ Known gaps

- A crash sends no `offline`, so open tabs can show a user as online for too long.
- The gateways get events from Redis pub/sub, not Kafka; Kafka only feeds the push worker.
- A deactivated user still gets push notifications.
- No rate limits: `429` and `4008 too_many_messages` are in the contract but never sent.
- A socket can miss a channel removal that happens while it connects ([CHAT-271](https://assist-team-m66z4ph1.atlassian.net/browse/CHAT-271)).

Full list: [known limitations](../../architecture/known-limitations.md#messages-and-live-events).

## Key code

| File | What is there |
|---|---|
| `app/main.py` | `create_app()`, and the lifespan: database, both Redis clients, the chatty-core client, the gateway and the `core.events` listener |
| `app/api/routers/messages.py` | Send, history, edit, delete, attachment links |
| `app/api/routers/reactions.py` | `PUT` and `DELETE …/reactions/{emoji}` |
| `app/api/routers/pins.py` | Pin, unpin, list pins |
| `app/api/routers/presence.py` | `GET /teams/{team_id}/presence` |
| `app/api/routers/sync.py` | `POST /sync`, `GET /channels/{id}/events` |
| `app/api/routers/search.py`, `app/services/search_service.py` | `GET /search`: readable channels, the cursor, the highlight |
| `app/api/routers/ws.py` | The WebSocket: token check, client messages, typing checks, deadlines, close codes |
| `app/services/message_service.py` | `MessageService`: send with attach and detach, edit, delete, history, attachment links, publish after commit |
| `app/services/reaction_service.py` | `ReactionService`: add and remove a reaction |
| `app/services/pin_service.py` | `PinService`: pin, unpin, list, the 100-pin limit |
| `app/services/presence_service.py` | `PresenceService`: the team check, who is online |
| `app/services/sync_service.py` | `SyncService`: sync and channel events, the gap checks |
| `app/services/gateway.py` | `Gateway`: one pub/sub connection, fan-out to local sockets, core events, presence, typing, watch |
| `app/services/connection_registry.py` | `ConnectionRegistry`: the open sockets of this process, by user, channel and team |
| `app/services/event_bus.py` | `EventBus`: channel events, live messages to one user and typing on Redis pub/sub |
| `app/services/access_service.py` | `AccessService`: the user, team, read, write and delete checks |
| `app/services/health_service.py` | `HealthService`: the database, Redis and background task checks |
| `app/repositories/` | Messages, reactions, pins, channel events, the seq counter, read state |
| `app/clients/core.py` | `CoreClient`: chatty-core's `/internal` API with the versioned Redis cache, attach and detach |
| `app/clients/storage.py` | `presign_attachment`: the download links, signed for the public MinIO address |
| `app/clients/presence.py` | `PresenceStore`: who is online and whose tab is on screen, in Redis |
| `app/api/routers/push.py`, `app/services/push_subscription_service.py` | Save and forget push subscriptions |
| `app/jobs/relay.py`, `app/services/outbox_relay.py` | The outbox relay to Kafka |
| `app/jobs/push_worker.py`, `app/services/push_service.py`, `app/clients/web_push.py` | The push worker: recipients, dedupe, Web Push |
| `app/clients/kafka.py` | The Kafka producer and consumer |
| `app/clients/core_events.py` | The `core.events` listener and cache clearing |
| `app/core/security.py` | `decode_access_token`: the RS256 check |
